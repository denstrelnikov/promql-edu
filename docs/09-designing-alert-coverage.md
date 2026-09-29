# 09. Проектирование покрытия алертами

## 9.1. Разбор входящей задачи

Выписать таблицу:

| Поле | Значение |
|------|----------|
| Git project | `order-hub` |
| Environment label | `prod1` |
| Компонент | CH / PG / Kafka / K8s deployment |
| Cluster service label | `order-hub-analytics-ch-cl1` |
| Datasource | vmselect / thanos (где есть series) |
| Пороги | явные или «как у группы X в repo» |
| Out of scope | staging, другие DB |

**Stop:** пока нет одного working query в Explore — не пишите YAML.

## 9.2. Матрица «компонент → алерты»

### K8s deployment

Availability, rollout-safe replicas, restarts, memory (± CPU).

### PostgreSQL

Exporters, pg_up, disk/cpu/ram node, connections, replication, pgbouncer, pgbackrest **по типам**.

### Kafka consumer

Lag sum, stalled/no members; broker down — только если в scope.

### ClickHouse

Exporter + node exporter, readonly, disk, optional replication.

## 9.3. MVP vs полный эталон

| MVP (узкая задача) | Полный эталон |
|--------------------|---------------|
| CH: up + disk | + readonly + replication + cpu |
| Kafka: lag | + stalled + broker |
| API: replicas | + memory + crashloop |

Согласуйте расширение отдельно, если в задаче только «availability + disk».

## 9.4. Пороги и `for` — ориентиры

| Сигнал | Warning | Critical | for |
|--------|---------|----------|-----|
| Disk | >80% | >90% | 15m / 5m |
| PG incr backup | — | >36h | 1h |
| Kafka lag | >10k | >50k | 15m / 5m |
| CH replication delay | >60s | >300s | 10m / 5m |
| K8s replicas | — | <spec | 5m |

Подстройка под SLO — после первых недель firing.

## 9.5. Имена и группы

- Одна задача на CH → одна группа `order-hub-clickhouse-infra`.
- Не создавать вторую группу с тем же service без причины.

## 9.6. Дублирование

Перед commit:

```bash
grep -n "alert: OrderHub" projects/order-hub/values-vmalert-vm.yaml
```

Не должно быть двух disk alerts на один cluster с разными именами.

## 9.7. Тексты для дежурного

**summary:** `[Компонент]: [проблема] — [{{ $labels.instance }}]`

**description:** что за кластер, текущее `{{ $value }}`, HAProxy, что смотреть первым.

## 9.8. Пример закрытия задачи

**Вход:** prod1, CH `order-hub-analytics-ch-cl1`, DB `order_hub_analytics`, health + disk.

**Выход:**

- Группа `order-hub-clickhouse-infra`, interval 30s.
- 5–8 rules (exporter, node, readonly, disk w/c).
- Комментарий с именем БД.
- MR и review у лида (глава 10).
- При необходимости — проверка по главе 11.

## 9.9. Anti-goals

- Test `vector(1)>0` в master.
- Алерты без `for` на шумных метриках.
- `min by (partition)` для lag без обоснования.
- Postgres rules в задаче ClickHouse.
