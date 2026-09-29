# 00. Наблюдаемость и алертинг с нуля

## 0.1. Зачем вообще метрики

**Метрика** — числовое измерение состояния системы во времени: «сколько запросов в секунду», «свободно байт на диске», «pod в Running или нет».

Три опоры observability (наблюдаемости):

| Опора | Что даёт | Алерты |
|-------|----------|--------|
| **Metrics** | агрегаты, тренды, пороги | основной материал этого курса |
| **Logs** | детали конкретного события | часто отдельный pipeline |
| **Traces** | путь запроса через сервисы | latency, errors (иногда через metrics) |

Алерт — **автоматическое правило**: если выражение над метриками истинно достаточно долго, система шлёт уведомление людям или в ITSM.

Цепочка, с которой вы работаете в типичном репозитории конфигурации:

```
Экспортёры / kube-state / приложение
        ↓ scrape или remote write
   TSDB (Prometheus, VictoriaMetrics, …)
        ↓ PromQL query
      vmalert (правила из Git)
        ↓ firing alert
   Alertmanager (маршруты из Git)
        ↓
 Telegram / Mattermost / webhook
```

Вы **не** пишете код exporter'а в каждой задаче — вы **подбираете PromQL** и **оформляете YAML**, где expr уже существующих метрик описывает проблему.

## 0.2. Prometheus в двух абзацах

**Prometheus** (и совместимые backend'ы) хранит **временные ряды**: последовательность `(timestamp, value)` для уникальной комбинации **имени метрики + labels**.

- **Pull-модель:** сервер периодически **запрашивает** HTTP endpoint `/metrics` (scrape).
- **Scrape interval** — как часто опрашивают (15s, 30s, 1m). От этого зависит «свежесть» данных и минимальное окно для `rate()`.
- **Retention** — сколько хранить историю (дни/месяцы).

**VictoriaMetrics (vmselect)** и **Thanos Querier** для вас ведут себя как **Prometheus API** для запросов: тот же PromQL (или расширенный MetricsQL), те же типы ответа. Алерт всё равно описывается expr, проверенным в Explore на **том же datasource**, что указан в values vmalert.

## 0.3. Имя метрики и labels

Пример строки в `/metrics` (формат text exposition):

```text
node_filesystem_avail_bytes{device="/dev/sda1",fstype="ext4",instance="ch-01.acme:9100",job="node",service="order-hub-analytics-ch-cl1",environment="prod1"} 4.8217731072e+10
```

Расшифровка:

| Часть | Значение |
|-------|----------|
| Имя | `node_filesystem_avail_bytes` |
| Labels | `device`, `fstype`, `instance`, `job`, `service`, `environment` |
| Значение | число (байты) |

**Один ряд (time series)** = имя + **полный** набор labels.  
Добавили label — это **другой** ряд.

В PromQL вы **фильтруете** labels в фигурных скобках:

```promql
node_filesystem_avail_bytes{service="order-hub-analytics-ch-cl1", job="node"}
```

## 0.4. Типы метрик (критично для алертов)

### Counter

- Монотонно **не убывает** (кроме рестарта процесса / reset).
- Имена часто с суффиксом `_total`, `_count`.
- Примеры: `http_requests_total`, `kube_pod_container_status_restarts_total`.
- **Нельзя** алертить «counter > 1000» без контекста — нужна **скорость** (`rate`) или **прирост** (`increase`).

### Gauge

- Может расти и падать.
- Примеры: `pg_up` (0/1), `node_memory_MemAvailable_bytes`, `kafka_consumergroup_lag`.
- Пороги напрямую: `lag > 10000`, `pg_up == 0`.

### Histogram

- Для распределений (latency, размеры).
- Порождает: `_bucket{le="..."}`, `_sum`, `_count`.
- Перцентили: `histogram_quantile(0.99, sum(rate(..._bucket[5m])) by (le, ...))`.

### Summary (реже)

- Квантили на стороне клиента; встречается реже histogram в Kubernetes-стеке.

## 0.5. Что такое «алерт» технически

Правило в vmalert:

| Поле | Роль |
|------|------|
| `expr` | PromQL; **непустой** результат = условие выполнено |
| `for` | сколько подряд evaluation'ов держать условие перед **firing** |
| `labels` | `severity`, `source`, `service`, … |
| `annotations` | тексты для человека (`summary`, `description`) |

Жизненный цикл:

```
inactive → (expr true) → pending → (for истёк) → firing
                ↓ expr false
            resolved (уведомление о восстановлении, если настроено)
```

- **Evaluation interval** группы (`interval: 30s`) — как часто vmalert пересчитывает expr.
- **`for: 5m`** — «не шуметь на кратковременный всплеск»: условие должно быть true **5 минут подряд** (с учётом логики vmalert/Prometheus).

Пустой результат expr = **OK** (нет алерта для этих series).

## 0.6. Instant query vs range query

| | Instant | Range |
|---|---------|--------|
| Вопрос | «Какое значение **сейчас**?» | «Как менялось **за окно**?» |
| В алертах | **да** | только внутри функций `[5m]` |
| В Grafana graph | точка / последнее | график |

Пример range **внутри** функции:

```promql
rate(http_requests_total[5m])
```

`[5m]` — **range vector**: все sample за последние 5 минут для каждого ряда.

## 0.7. Datasource: где лежат ваши series

Один и тот же сервис может отдавать метрики в **разные** хранилища:

| Площадка | Типичный datasource в Grafana | Файл values |
|----------|------------------------------|-------------|
| K8s в облаке | Thanos | `values-vmalert-thanos.yaml` |
| K8s / VM на «своей» площадке | vmselect | `values-vmalert-vm.yaml` |

**Правило:** алерт живёт там, **где Explore показывает series** для ваших labels.  
Неверный datasource → вечный «No data» → алерт **никогда** не firing (ложное спокойствие).

## 0.8. Вымышленный контекст Acme / order-hub

Дальше по документу:

| Сущность | Имя |
|----------|-----|
| Git-проект | `order-hub` |
| K8s namespace prod | `order-hub-prod1` |
| API deployment | `order-hub-api` |
| PG cluster label `service` | `order-hub-pgsql-cl1` |
| Kafka cluster | `shared-events-kfk-cl1` |
| ClickHouse cluster | `order-hub-analytics-ch-cl1` |
| БД ClickHouse | `order_hub_analytics` (в текстах, не обязательно в PromQL) |
| Окружение | `environment="prod1"` |

## 0.9. Инструменты на вашей стороне

| Инструмент | Для чего |
|------------|----------|
| **Grafana Explore** | отладка PromQL, проверка labels |
| **Git + GitLab MR** | изменение YAML, CI |
| **kubectl port-forward** | UI vmalert / Alertmanager (если есть доступ к кластеру) |

Деплой конфигурации после merge в `master` в типичном setup **автоматизирован** (helm upgrade из CI или отдельного job) — вручную kubectl/helm нужен только для **отладки**, не для каждого MR.

## 0.10. Карта документа

| Вопрос | Глава |
|--------|-------|
| Как написать первый selector? | 01 |
| Как считать RPS и рестарты? | 02 |
| Как подавить алерт при деплое? | 03 |
| Как не сломать YAML? | 04 |
| Куда положить файл в репо? | 05 |
| Почему алерт в `drop`? | 06, 11 |
| MR и конфликт в master? | 10 |

Следующая глава: [01-promql-selectors-and-types.md](01-promql-selectors-and-types.md).
