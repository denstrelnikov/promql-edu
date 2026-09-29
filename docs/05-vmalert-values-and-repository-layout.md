# 05. Helm values vmalert и layout репозитория

Типичный GitOps-репозиторий алертинга (монorepo vmalert + Alertmanager) хранит **декларативно**:

- правила per-project vmalert;
- общий Alertmanager;
- CI, который **lint'ит** YAML на MR и master;
- автоматическое применение в K8s после merge (без ручного шага в обычном flow).

## 5.1. Дерево каталогов

```text
repo-root/
├── .gitlab-ci.yml
├── shared/
│   ├── values-alertmanager.yaml    # routes, receivers
│   └── values-keephq.yaml          # ITSM webhook (если используется)
└── projects/
    └── order-hub/
        ├── values-vmalert-vm.yaml
        └── values-vmalert-thanos.yaml   # опционально
```

| Путь | Содержимое |
|------|------------|
| `projects/<name>/` | один «продуктовый» project label |
| `values-vmalert-vm.yaml` | datasource vmselect |
| `values-vmalert-thanos.yaml` | datasource Thanos |
| `shared/` | компоненты без project label |

## 5.2. Каркас values vmalert

```yaml
server:
  replicaCount: 2

  datasource:
    url: "http://vmselect-primary.acme.example:8481/select/0/prometheus"

  remoteRead:
    url: "http://vmselect-secondary.acme.example:8481/select/0/prometheus"

  notifier:
    url: "http://alertmanager.external-monitoring.svc.cluster.local:9093"

  extraArgs:
    httpListenAddr: :8880
    external.label: "project=order-hub"

  config:
    alerts:
      groups:
        # ... ваши группы ...

serviceMonitor:
  enabled: true
```

### `datasource.url`

Откуда vmalert **читает** метрики для expr. Должен совпадать с Grafana Explore при проверке.

### `remoteRead.url`

Второй endpoint (DR / другой ДЦ) — особенности chart; в battle repos часто заполнен парой vmselect.

### `notifier.url`

Куда vmalert шлёт firing alerts — **Alertmanager**.

### `external.label`

**Критично:** строка вида `project=order-hub` добавляется ко **всем** алертам этого release.  
Alertmanager матчит `project="order-hub"` → receiver.

**Не** добавляйте `project:` в `labels` каждого правила — риск рассинхрона с route.

## 5.3. Именование Helm release (логика)

| Файл | Release (пример) |
|------|------------------|
| `values-vmalert-vm.yaml` | `vmalert-order-hub-vm` |
| `values-vmalert-thanos.yaml` | `vmalert-order-hub-thanos` |

Один project в git может иметь **два** pod vmalert — разные datasource.

## 5.4. Выбор vm vs thanos

| Вопрос | Действие |
|--------|----------|
| Pod'ы сервиса в облачном K8s, метрики только в Thanos | `values-vmalert-thanos.yaml`, `source: thanos` |
| PG/Kafka/CH на VM, метрики в vmselect | `values-vmalert-vm.yaml`, `source: vmselect` |
| API в YC, PG на VM | **два файла**, два release |

## 5.5. Группировка rules внутри файла

Рекомендуемые группы (имена — пример):

| group name | Содержание |
|------------|------------|
| `order-hub-api-k8s` | deployment api |
| `order-hub-worker-k8s` | workers |
| `order-hub-postgres-infra` | node, exporters, disk |
| `order-hub-postgres-database` | connections, replication |
| `order-hub-postgres-pgbouncer` | pools |
| `order-hub-postgres-backup` | pgbackrest |
| `order-hub-kafka` | lag, stalled |
| `order-hub-clickhouse-infra` | CH health, disk |

Одна **задача Jira** не обязана создавать все группы — добавляете **одну** группу или дополняете существующую.

## 5.6. Изменение shared vs projects

| Изменение | Файл |
|-----------|------|
| Новый алерт | `projects/.../values-vmalert-*.yaml` |
| Новый Telegram канал для project | `shared/values-alertmanager.yaml` |
| Новый project с нуля | папка `projects/` + route + receiver в shared |

MR с shared требует более внимательного review (секреты webhook/token — masked variables в GitLab, не в git).

## 5.7. CI validate (пример: external-monitoring)

Pipeline stage **validate**:

```yaml
helm lint alertmanager -f shared/values-alertmanager.yaml
helm lint keep -f shared/values-keephq.yaml
for f in projects/*/values-vmalert-*.yaml; do
  helm lint victoria-metrics-alert -f "$f"
done
```

Триггеры: **merge request** и **master**.

**CI не выполняет PromQL** против live metrics — только структура chart + values.

После merge в master деплой конфигурации в кластер выполняется **автоматически** (отдельный процесс/org-specific); в MR вам нужно **зелёный validate** и code review.

## 5.8. Новый project — минимальный набор изменений

1. `mkdir projects/new-service`
2. Скопировать values с похожего сервиса, поменять `external.label`, datasource URL при необходимости.
3. В `shared/values-alertmanager.yaml`:
   - route `project="new-service"` → receiver `new-service-all`
   - receiver с telegram_configs / mattermost_configs
4. MR, lint, merge.

## 5.9. Связь с examples/

Файл [examples/order-hub-values-vmalert-vm.yaml](../examples/order-hub-values-vmalert-vm.yaml) — сокращённый, но **структурно корректный** образец.

---

Далее: Alertmanager — [06-alertmanager-routing.md](06-alertmanager-routing.md).
