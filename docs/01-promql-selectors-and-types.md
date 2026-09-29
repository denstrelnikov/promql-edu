# 01. PromQL: selectors, типы результата, простые выражения

PromQL (Prometheus Query Language) — язык запросов к временным рядам. Эта глава — фундамент: что возвращает запрос, как выбирать ряды, как фильтровать и сравнивать.

## 1.1. Четыре типа результата PromQL

| Тип | Описание | Пример использования |
|-----|----------|----------------------|
| **Scalar** | одно число | редко в финальном expr алерта |
| **String** | строка | редко |
| **Instant vector** | набор пар (labels → value) на один момент времени | **основной результат алерта** |
| **Range vector** | набор пар (labels → список значений за окно) | аргумент `rate(...[5m])` |

Алерт vmalert ожидает, что expr вернёт **instant vector** (возможно пустой).  
Каждый элемент vector = отдельная потенциальная «строчка» firing (Alertmanager может **group** их в одно сообщение).

## 1.2. Selector — выбор time series

### Только имя метрики

```promql
pg_up
```

Вернёт **все** ряды `pg_up` (может быть тысячи). В Explore — первый шаг «метрика существует?».

### С фильтром по labels

```promql
pg_up{service="order-hub-pgsql-cl1", environment="prod1"}
```

Все label matchers **одновременно** (логическое AND).

### Операторы сравнения для labels

| Синтаксис | Значение |
|-----------|----------|
| `label="value"` | равно |
| `label!="value"` | не равно |
| `label=~"regex"` | регулярное выражение (RE2) |
| `label!~"regex"` | не match |

Примеры:

```promql
kube_pod_container_status_running{namespace="order-hub-prod1", container="order-hub-api"}

kube_deployment_spec_replicas{cluster=~"k8s-prd-alpha|k8s-prd-beta"}

node_filesystem_avail_bytes{fstype!~"tmpfs|fuse.*"}
```

### Regex для instance (инфра-кластеры)

Шаблон для всех нод кластера:

```promql
instance=~"order-hub-pgsql-cl1-[0-9]+.*:9100"
```

- `[0-9]+` — номер ноды `01`, `02`, …
- `.*` — произвольный суффикс (FQDN)
- `:9100` — node exporter

**Не** пишите `instance="host:9100"` одной строкой, если не проверили точный формат в Explore.

### Пустой selector `{}`

```promql
up{}
```

Все ряды `up` — тяжёлый запрос; используйте для разведки, не в prod-алертах.

## 1.3. Метки `__name__`

Скрытый label — имя метрики:

```promql
{__name__=~"kafka.*lag.*", service="shared-events-kfk-cl1"}
```

Помогает найти «как называется lag», если в документации другое имя.

## 1.4. Сравнение с числом (filtering)

Binary comparison **между instant vector и scalar** отфильтровывает ряды:

```promql
pg_up{service="order-hub-pgsql-cl1"} == 0
```

Останутся только ряды, где значение **0** (PostgreSQL down с точки зрения exporter).

```promql
kafka_consumergroup_lag{consumergroup="kafka_consumer_order_hub_prod"} > 10000
```

Значение comparison **не** меняет value в результате для alerting — остаётся исходное значение lag (важно для `{{ $value }}` в шаблоне).

### Диапазоны

```promql
node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes < 0.1
```

(лучше явно `* 100 > 90` для «процентов занято» — читаемее в алерте).

## 1.5. Арифметика между instant vectors

Если у двух vector **одинаковый набор labels** (попарно), операция поэлементная:

```promql
node_filesystem_avail_bytes{...} / node_filesystem_size_bytes{...}
```

Если labels **не совпадают** — нужно выравнивание (`on`, `ignoring`) — глава 03.

Допустимые операции: `+`, `-`, `*`, `/`, `%`, `^`.

## 1.6. Функции, меняющие labels (preview)

| Функция | Эффект |
|---------|--------|
| `abs(v)` | модуль |
| `clamp_min(v, 1)` | не меньше 1 (защита от деления на 0) |
| `clamp_max(v, 100)` | не больше 100 |
| `round(v, 0.01)` | округление |

Пример для алерта памяти pod:

```promql
clamp_min(kube_pod_container_resource_limits{resource="memory", unit="byte"}, 1)
```

## 1.7. Пошаговая сборка expr (метод работы)

1. **Explore**, datasource = будущий vmalert.
2. Запрос `{service="..."}` или `up{service="..."}` — есть series?
3. Добавляйте по одному label, смотрите, не стало `No data`.
4. Добавьте математику / aggregation (глава 02).
5. Добавьте порог `> X`.
6. Time range **24h** — были ли ложные срабатывания на истории?
7. Скопируйте в YAML `expr: >-`.

## 1.8. Частые синтаксические ошибки

| Ошибка | Пример | Исправление |
|--------|--------|-------------|
| Две запятые подряд | `job="node", ,` | убрать пустой matcher |
| Trailing comma | `job="clickhouse",}` | убрать запятую перед `}` |
| Кавычки | `service='foo'` | в PromQL строки в **двойных** `"` |
| Regex без anchor | `instance=~"9100"` | сузить `.*:9100` + service |
| `=` вместо `=~` для списка | `cluster="a\|b"` | `cluster=~"a\|b"` |

YAML **дополнительно** ломает expr, если не используете `>-` и не экранируете спецсимволы — глава 04.

## 1.9. Упражнения (мысленно)

1. Выберите все `up` для job postgres кластера `order-hub-pgsql-cl1`, prod1, порт 9187.  
   → см. [appendix-a](appendix-a-labels-and-naming.md)

2. Почему `node_filesystem_avail_bytes{service="..."} > 80` **не** «80% диска»?  
   → потому что значение в **байтах**, не процентах; нужна формула с `size_bytes`.

3. Сколько рядов вернёт `pg_up{service="order-hub-pgsql-cl1"}`?  
   → по одному на каждую уникальную комбинацию **остальных** labels (обычно instance/replica).

## 1.10. Полный walkthrough: «PostgreSQL недоступен»

**Шаг 1.** Найти metric:

```promql
pg_up
```

**Шаг 2.** Сузить:

```promql
pg_up{service="order-hub-pgsql-cl1"}
```

**Шаг 3.** Добавить env и instance:

```promql
pg_up{
  service="order-hub-pgsql-cl1",
  environment="prod1",
  instance=~"order-hub-pgsql-cl1-[0-9]+.*:9187"
}
```

**Шаг 4.** Условие алерта:

```promql
pg_up{
  service="order-hub-pgsql-cl1",
  environment="prod1",
  instance=~"order-hub-pgsql-cl1-[0-9]+.*:9187"
} == 0
```

**Шаг 5.** Проверить в Explore: при штатной работе — **empty** (нет рядов с 0). При падении — one or more series.

Аналог для scrape:

```promql
up{job="postgres", service="order-hub-pgsql-cl1", ...} == 0
```

`up` и `pg_up` **дополняют** друг друга: target не scrape'ится vs postgres отвечает exporter'у.

## 1.11. Literal numbers и единицы

PromQL не хранит «проценты» в типе — вы **вычисляете** число и сравниваете:

```promql
... > 80
```

для «80% disk».  
Для байт:

```promql
node_filesystem_avail_bytes < 10 * 1024 * 1024 * 1024
```

(10 GiB free — пример абсолютного порога, хуже чем %).

## 1.12. Несколько рядов = несколько алертов

Expr вернул 3 instance с `pg_up==0` → vmalert создаст **3** alert (до group_by в AM).  
Если нужен **один** алерт на кластер — оберните:

```promql
count(pg_up{service="order-hub-pgsql-cl1"} == 0) > 0
```

или `max(...)`, или `sum(...)`, осознанно теряя instance в labels (в summary тогда не будет `$labels.instance` без `by`).

---

Далее: агрегации и counter — [02-promql-aggregations-and-functions.md](02-promql-aggregations-and-functions.md).
