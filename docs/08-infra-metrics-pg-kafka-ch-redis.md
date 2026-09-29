# 08. Инфраструктурные метрики: PostgreSQL, Kafka, ClickHouse, Redis

VM-кластеры с unified labels `service`, `environment`, `job`, `instance`.  
Файл values: **`values-vmalert-vm.yaml`**, `source: vmselect`.

## 8.1. PostgreSQL — слои мониторинга

| Слой | job | Порт | Метрики |
|------|-----|------|---------|
| OS | node | 9100 | disk, cpu, ram |
| Postgres | postgres | 9187 | `pg_up`, connections, replication |
| PgBouncer | pgbouncer | 9127 | pools, wait |
| pgBackRest | pgbackrest | 9854 | backup age, WAL |

### Exporter down

```promql
up{
  service="order-hub-pgsql-cl1",
  environment="prod1",
  job="postgres",
  instance=~"order-hub-pgsql-cl1-[0-9]+.*:9187"
} == 0
```

### pg_up

```promql
pg_up{
  service="order-hub-pgsql-cl1",
  environment="prod1",
  instance=~"order-hub-pgsql-cl1-[0-9]+.*:9187"
} == 0
```

Разница: `up` — scrape target; `pg_up` — логика exporter'а (может быть 0 при живом scrape).

### Connections

```promql
sum by (instance) (
  pg_stat_activity_count{service="order-hub-pgsql-cl1", state="active"}
)
/
sum by (instance) (
  pg_settings_max_connections{service="order-hub-pgsql-cl1"}
) > 0.8
```

### Replication lag

```promql
pg_replication_lag_seconds{
  service="order-hub-pgsql-cl1",
  environment="prod1"
} > 300
```

Имена метрик зависят от версии exporter — проверить в Explore.

## 8.2. pgBackRest — три типа backup

| backup_type | Порог (сек) | for |
|-------------|-------------|-----|
| incr | 129600 (36h) | 1h |
| full | 604800 (7d) | 1h |
| diff | 608400 (7d+1h) | 1h |

**Отдельное правило на каждый тип.**  
Не объединяйте incr и diff с порогом 36h.

```promql
pgbackrest_backup_since_last_completion_seconds{
  service="order-hub-pgsql-cl1",
  environment="prod1",
  backup_type="incr"
} > 129600
```

### WAL archive

```promql
pgbackrest_wal_archive_status{
  service="order-hub-pgsql-cl1",
  environment="prod1"
} == 0
```

### Check / stanza errors

Зависит от exporter — `pgbackrest_backup_error`, `pgbackrest_stanza_status` и т.д. (смотреть `/metrics`).

## 8.3. Kafka

### Broker availability

```promql
up{
  service="shared-events-kfk-cl1",
  environment="prod1",
  job="kafka",
  instance=~"shared-events-kfk-cl1-[0-9]+.*"
} == 0
```

Порт в instance может быть JMX exporter или kafka_exporter — уточнить в Explore.

### Consumer lag (project-specific)

Алерт лежит в **values project-потребителя** (`order-hub`), метрики — с **общего** кластера:

```promql
sum(
  kafka_consumergroup_lag{
    environment="prod1",
    service="shared-events-kfk-cl1",
    topic="orders-status-changed",
    consumergroup="kafka_consumer_order_hub_prod"
  }
) > 10000
```

`for: 15m` — lag не отрабатывается.

### Stalled consumer

Паттерн (идея): lag > X **и** rate offset ≈ 0 **и** members > 0.  
Точный expr копируют из уже работающей группы kafka **того же** git-репозитория — labels сильно зависят от exporter.

### Under-replicated partitions

```promql
kafka_cluster_partition_under_replicated_partition{
  service="shared-events-kfk-cl1"
} > 0
```

## 8.4. ClickHouse

Кластер: `order-hub-analytics-ch-cl1`.  
БД `order_hub_analytics` — в comment/description группы.

### Минимальный health + disk набор

| Alert | Expr core |
|-------|-----------|
| ExporterDown | `up{job="clickhouse", instance=~"...:9116"} == 0` |
| NodeExporterDown | `up{job="node", instance=~"...:9100"} == 0` |
| ReadonlyReplica | `clickhouse_readonly_replica{...} == 1` |
| DiskWarning | node FS > 80% |
| DiskCritical | node FS > 90% |

### Replication (расширение)

```promql
ClickHouseAsyncMetrics_ReplicasMaxAbsoluteDelay{
  service="order-hub-analytics-ch-cl1",
  environment="prod1",
  instance=~"order-hub-analytics-ch-cl1-[0-9]+.*:9116"
} > 60
```

```promql
ClickHouseAsyncMetrics_ReplicasSumQueueSize{...} > 100
```

```promql
sum by (instance) (
  rate(ClickHouseProfileEvents_ReplicatedPartFailedFetches{...}[5m])
) > 0
```

### Специфичные exporter metrics

| Метрика | Смысл |
|---------|--------|
| `clickhouse_readonly_replica` | replica read-only |
| `ClickHouseMetrics_*` | внутренние счётчики |
| `ClickHouseAsyncMetrics_*` | async репликация |

Все CH app metrics — job `clickhouse`, port **9116** в типичной схеме.

Disk — **только** через `job="node"` и `:9100`, не путать с CH port.

## 8.5. Redis

```promql
redis_up{service="order-hub-cache-redis-cl1"} == 0
```

```promql
redis_memory_used_bytes / redis_memory_max_bytes > 0.9
```

```promql
redis_connected_clients / redis_config_maxclients > 0.9
```

Label `service` зависит от регистрации targets.

## 8.6. HAProxy / entrypoint

В annotations, не в PromQL:

```text
order-hub-pgsql-cl1-haproxy.db.acme.example
order-hub-analytics-ch-cl1-haproxy.db.acme.example
```

## 8.7. Scope задачи

Задача «только ClickHouse» → **не** добавлять postgres disk в тот же MR.  
Задача «Kafka lag topic X» → не дублировать broker disk без запроса.

---

Далее: проектирование покрытия — [09-designing-alert-coverage.md](09-designing-alert-coverage.md).
