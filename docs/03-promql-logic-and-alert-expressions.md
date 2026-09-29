# 03. PromQL: and, or, unless, on(), group_left — выражения для алертов

## 3.1. Логические операторы между vectors

PromQL **не** имеет `AND`/`OR` для boolean — используются **set operations** на instant vectors.

### `and`

Оставляет элементы **левого** vector, для которых есть совпадающий **справа** (по default — все labels):

```promql
high_error_rate and on(namespace) kube_deployment_spec_replicas{replicas="3"}
```

### `or`

Объединение множеств (может дублировать alert labels — используйте редко).

### `unless` — подавление при rollout K8s

**Левое** expr минус те series, где **правое** expr имеет совпадение:

```promql
(
  max by (cluster, namespace, deployment) (
    kube_deployment_status_replicas_available{
      namespace="order-hub-prod1",
      deployment="order-hub-api"
    }
  )
  <
  max by (cluster, namespace, deployment) (
    kube_deployment_spec_replicas{
      namespace="order-hub-prod1",
      deployment="order-hub-api"
    }
  )
)
unless on(cluster, namespace, deployment) (
  max by (cluster, namespace, deployment) (
    kube_deployment_status_observed_generation{
      namespace="order-hub-prod1",
      deployment="order-hub-api"
    }
    != kube_deployment_metadata_generation{
      namespace="order-hub-prod1",
      deployment="order-hub-api"
    }
  )
)
```

Пока `observed_generation != metadata_generation`, deployment **обновляется** — алерт «мало реплик» **подавлен**.

### Почему `on(cluster, namespace, deployment)`

Без `on()` matching идёт по **полному** набору labels. Generation-metrics и replica-metrics могут иметь разные лишние labels → join не сработает → `unless` **не подавит** → ложные алерты при деплое.

Для API в **двух** кластерах всегда включайте **`cluster`** в `on(...)`.

## 3.2. on() и ignoring()

### `on(label1, label2)`

Указать, **по каким labels** матчить левую и правую часть бинарной операции.

```promql
A / on(instance) B
```

### `ignoring(label1)`

Матч по всем labels, **кроме** перечисленных.

```promql
A / ignoring(cpu) B
```

## 3.3. group_left / group_right

Когда с одной стороны **больше** labels, чем с другой, и нужно «подтянуть» labels с одной стороны:

```promql
sum by (pod) (container_memory_working_set_bytes{...})
/ on(namespace, pod) group_left
kube_pod_container_resource_limits{resource="memory", ...}
```

В алертах чаще заранее `sum by (namespace, pod, container)` с обеих сторон, чтобы не усложнять.

## 3.4. Сравнение двух metrics с разными labels — типовой паттерн

**Плохо:** делить без выравнивания.

**Хорошо:** явные `by` и одинаковые labels:

```promql
sum by (namespace, pod, container) (
  container_memory_working_set_bytes{namespace="order-hub-prod1", container="order-hub-api"}
)
/
clamp_min(
  sum by (namespace, pod, container) (
    kube_pod_container_resource_limits{
      namespace="order-hub-prod1",
      container="order-hub-api",
      resource="memory",
      unit="byte"
    }
  ),
  1
) > 0.90
```

## 3.5. bool modifier (редко)

```promql
(A > B) bool
```

Превращает comparison в 0/1 — для recording rules; в alerting обычно не нужно.

## 3.6. Subquery (запрос в запросе)

```promql
max_over_time(
  (sum(kafka_consumergroup_lag{...}) > 10000)[1h:5m]
)
```

«Был ли lag > 10k на каждом шаге 5m внутри часа» — продвинутый кейс; для большинства задач достаточно `for: 15m` в правиле.

## 3.7. VictoriaMetrics MetricsQL

vmselect поддерживает PromQL и расширения (rollup, `default`, …). Если expr из чистого Prometheus ведёт себя иначе:

- документация: https://docs.victoriametrics.com/metricsql/
- первый шаг — упростить до canonical PromQL

95% правил в values-файлах — чистый PromQL.

## 3.8. Шаблон «готового» алерта availability

```promql
up{
  service="TARGET_CLUSTER",
  environment="prod1",
  job="JOB_NAME",
  instance=~"TARGET_CLUSTER-[0-9]+.*:PORT"
} == 0
```

Замените `JOB_NAME` / `PORT` по appendix-a.

## 3.9. Антипаттерны в expr алертов

| Антипаттерн | Почему плохо |
|-------------|--------------|
| `rate(gauge[5m])` | gauge не counter |
| Алерт без порога | vector всегда firing |
| `min()` lag по partition | один «хороший» partition |
| Слишком широкий selector | алерт на чужой service |
| `backup_type=~"incr\|diff"` + порог 36h | weekly diff всегда stale |
| Нет `unless` на lifecycle | шум при каждом deploy |

## 3.10. Чеклист перед вставкой expr в YAML

- [ ] Instant vector, не range.
- [ ] Порог или `== 0` явно.
- [ ] Labels сужены до одного env / cluster.
- [ ] Counter → `rate`/`increase`.
- [ ] K8s availability → `unless` rollout.
- [ ] Explore 24h на **правильном** datasource.

---

Далее: YAML — [04-yaml-from-zero-to-alert-rules.md](04-yaml-from-zero-to-alert-rules.md).
