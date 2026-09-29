# 07. Алерты Kubernetes

Метрики K8s приходят из **kube-state-metrics** (объекты API: Deployment, Pod, …) и **cAdvisor/kubelet** (CPU, memory контейнеров).  
Labels: `cluster`, `namespace`, `pod`, `deployment`, `container`.

## 7.1. Подготовка данных для expr

Перед написанием алерта соберите в Explore:

```promql
kube_pod_container_info{namespace="order-hub-prod1", container=~"order-hub.*"}
```

Уточните **точное** имя `container` (часто = deployment, но бывают sidecar).

```promql
kube_deployment_spec_replicas{namespace="order-hub-prod1"}
```

Список deployment'ов в namespace.

## 7.2. Replicas available < spec

См. полный expr с `unless` в [03-promql-logic-and-alert-expressions.md](03-promql-logic-and-alert-expressions.md).

Параметры:

- `for: 5m` — типично для availability.
- `severity: critical`.

## 7.3. Ноль running pods

```yaml
expr: >-
  sum by (cluster, namespace, container) (
    kube_pod_container_status_running{
      namespace="order-hub-prod1",
      cluster=~"k8s-prd-alpha|k8s-prd-beta",
      container="order-hub-api"
    }
  ) == 0
  unless on(cluster, namespace) (
    max by (cluster, namespace) (
      kube_deployment_status_observed_generation{
        namespace="order-hub-prod1",
        cluster=~"k8s-prd-alpha|k8s-prd-beta",
        deployment="order-hub-api"
      }
      != kube_deployment_metadata_generation{
        namespace="order-hub-prod1",
        cluster=~"k8s-prd-alpha|k8s-prd-beta",
        deployment="order-hub-api"
      }
    )
  )
```

## 7.4. Restarts

| Алерт | Expr idea | severity |
|-------|-----------|----------|
| CrashLoop | `increase(restarts[15m]) > 5` | critical |
| Любой рестарт | `increase(restarts[10m]) > 0` | warning |

Всегда с `unless` rollout для связанного deployment.

## 7.5. Memory vs limit

```promql
sum by (cluster, namespace, pod, container) (
  container_memory_working_set_bytes{
    namespace="order-hub-prod1",
    container="order-hub-api"
  }
)
/
clamp_min(
  kube_pod_container_resource_limits{
    namespace="order-hub-prod1",
    container="order-hub-api",
    resource="memory",
    unit="byte"
  },
  1
) > 0.90
```

`working_set_bytes` — то, что реально давит на node (ближе к OOM killer).

## 7.6. CPU

Средняя загрузка от limit (если limits заданы):

```promql
sum by (namespace, pod, container) (
  rate(container_cpu_usage_seconds_total{container="order-hub-api", ...}[5m])
)
/
clamp_min(
  kube_pod_container_resource_limits{resource="cpu", ...},
  0.001
) > 0.85
```

Или абсолютный CPU без limit — реже для алертов.

## 7.7. Pod not ready / phase

```promql
kube_pod_status_phase{phase="Pending", namespace="order-hub-prod1", pod=~"order-hub-api.*"} == 1
```

Используйте осторожно — Pending при schedule может быть кратковременным; нужен `for`.

## 7.8. HPA (если есть)

```promql
kube_horizontalpodautoscaler_status_current_replicas
<
kube_horizontalpodautoscaler_spec_min_replicas
```

Специфично для autoscaling — копировать из соседнего сервиса.

## 7.9. Multi-cluster

Prod в двух cluster — **один** алерт с `cluster=~"a|b"` и `by (cluster, ...)` в summary:

```yaml
summary: "order-hub-api [{{ $labels.cluster }}]: мало реплик"
```

`unless on(cluster, namespace)` — **обязательно** cluster в `on()`.

## 7.10. Staging isolation

Namespace `order-hub-stg1` **не** включать в prod expr.  
Один values-файл prod — явный whitelist namespace.

## 7.11. HTTP / business metrics

Имена application metrics зависят от фреймворка. Паттерн:

```promql
sum(rate(http_requests_total{namespace="order-hub-prod1", status=~"5.."}[5m]))
/
sum(rate(http_requests_total{namespace="order-hub-prod1"}[5m])) > 0.05
```

Guard: знаменатель `> 0` через `and` или `clamp_min` на знаменатель.

Datasource: where app exports — thanos или vmselect.

## 7.12. Минимальный набор для stateless API

| # | Alert | for |
|---|-------|-----|
| 1 | ReplicasUnavailable | 5m |
| 2 | PodsNotRunning | 5m |
| 3 | CrashLooping | 5m |
| 4 | PodRestarted | 5m |
| 5 | MemoryNearLimit | 5m |

Workers/consumers — те же паттерны + Kafka lag (глава 08).
