# 04. YAML с нуля до правил алертов

Конфигурация алертов хранится в **YAML** внутри Helm values. Эта глава — синтаксис YAML и точная форма блока `groups` / `rules`.

## 4.1. YAML в двух словах

YAML описывает **дерево** из:

- **mapping** (словарь): `key: value`
- **sequence** (список): `- item`

Отступы **пробелами** (обычно 2), **не табы**.  
Sibling-элементы — **одинаковый** отступ.

```yaml
server:
  name: vmalert-order-hub
  config:
    alerts:
      groups:
        - name: example
          rules:
            - alert: Foo
              expr: up == 1
```

## 4.2. Типы scalar

| Запись | Тип |
|--------|-----|
| `123` | число |
| `true` / `false` | bool |
| `prod1` | строка (без кавычек OK) |
| `"8080:8080"` | строка с двоеточием — лучше в кавычках |
| `>-\n  line1\n  line2` | folded string (переносы → пробел) |

### Folded block `>-` для PromQL

```yaml
expr: >-
  sum(
    kafka_consumergroup_lag{
      service="shared-events-kfk-cl1"
    }
  ) > 10000
```

PromQL становится **одной логической строкой** — так ожидает vmalert.

### Literal block `|`

Сохраняет переводы строк — для шаблонов сообщений, не для expr.

## 4.3. Структура одного правила

```yaml
- alert: OrderHubKafkaConsumerLagHigh
  expr: >-
    sum(kafka_consumergroup_lag{...}) > 10000
  for: 15m
  labels:
    severity: warning
    source: vmselect
    service: order-hub-kafka
  annotations:
    summary: "Kafka: высокий lag — {{ $labels.topic }}"
    description: >-
      Consumer group kafka_consumer_order_hub_prod не успевает.
      Текущий lag: {{ $value | humanize }}.
      Topic: orders-status-changed.
```

| Поле | Обязательность | Заметки |
|------|----------------|---------|
| `alert` | да | уникальное имя в пределах vmalert instance |
| `expr` | да | PromQL |
| `for` | рекомендуется | `0` = сразу firing |
| `labels` | да (практически) | severity, source |
| `annotations` | да | summary + description |

## 4.4. Структура группы

```yaml
groups:
  - name: order-hub-clickhouse-infra
    interval: 30s
    rules:
      - alert: ...
      - alert: ...
```

| Поле группы | Смысл |
|-------------|--------|
| `name` | логическое имя (в UI vmalert) |
| `interval` | период evaluation всех rules в группе |
| `rules` | список алертов |

**Каждый** алерт должен быть **внутри** `rules:` какой-то группы.

### Антипример (структурная ошибка)

```yaml
  - name: order-hub-kafka
    rules:
      - alert: LagHigh
        expr: ...

  - alert: ClickhouseDown    # ← элемент groups БЕЗ name/rules
    expr: ...
```

Второй блок — не группа и не правило в группе; helm lint может пройти (YAML валиден), vmalert/chart может **игнорировать** или падать.

## 4.5. Annotations и шаблоны

Доступно в Alertmanager (не в PromQL):

| Переменная | Содержимое |
|------------|------------|
| `$labels.<name>` | label из firing alert |
| `$value` | значение sample |
| `$externalLabels` | labels с vmalert (в т.ч. project) |

Фильтры: `humanize`, `humanizeDuration`, `humanizePercentage`, `title`.

**HAProxy / runbook** — только **внутри** `description`:

```yaml
description: >-
  Instance {{ $labels.instance }} недоступна.
  HAProxy: order-hub-pgsql-cl1-haproxy.db.acme.example.
```

Строка `HAProxy:` с отступом как у **ключа** YAML — **ошибка** (отдельное поле, не текст алерта).

## 4.6. labels: что писать руками

| Label | Пример | Кто задаёт |
|-------|--------|------------|
| `project` | order-hub | **vmalert** `external.label` — **не дублировать** |
| `severity` | warning / critical | автор правила |
| `source` | vmselect / thanos | автор (= datasource) |
| `service` | order-hub-pgsql-cl1 | автор (кластер или компонент) |

## 4.7. Именование alert

PascalCase, префикс проекта:

```
OrderHub + Clickhouse + DiskUsageCritical
→ OrderHubClickhouseDiskUsageCritical
```

Не `order_hub_clickhouse_down`, не `api-order-hub` в имени алерта.

## 4.8. Комментарии

```yaml
        # ClickHouse order-hub-analytics-ch-cl1, database order_hub_analytics.
        - name: order-hub-clickhouse-infra
```

Комментарий **перед** `- name:` на том же уровне отступа, что и группы.

## 4.9. Валидация YAML локально

```bash
python3 -c "import yaml,sys; yaml.safe_load(open(sys.argv[1]))" projects/foo/values-vmalert-vm.yaml
```

Или `helm lint` (как в CI) — глава 10.

## 4.10. Типичные ошибки YAML + PromQL

| Симптом | Причина |
|---------|---------|
| HAProxy в UI не виден | вынесен из description |
| PromQL parse error | `, ,` в expr |
| Алерт не появляется в UI vmalert | не в `rules` |
| Двойные уведомления | два одинаковых алерта в разных группах |

---

Далее: layout репозитория — [05-vmalert-values-and-repository-layout.md](05-vmalert-values-and-repository-layout.md).
