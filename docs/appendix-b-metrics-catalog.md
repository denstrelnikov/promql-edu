# Appendix B. Каталог метрик для алертов

Версии exporter'ов меняют имена — финальная проверка всегда в Explore.

## B.1. Availability

| Metric | Expr |
|--------|------|
| up | `up{job="...", instance=~"..."} == 0` |
| pg_up | `pg_up{...} == 0` |
| redis_up | `redis_up == 0` |

## B.2. node_exporter

| Metric | Use |
|--------|-----|
| node_filesystem_avail_bytes | disk |
| node_filesystem_size_bytes | disk |
| node_cpu_seconds_total | CPU (rate, mode=idle) |
| node_memory_MemAvailable_bytes | RAM |
| node_memory_MemTotal_bytes | RAM |
| node_load1 | optional |

## B.3. Kubernetes

| Metric | Use |
|--------|-----|
| kube_deployment_status_replicas_available | replicas |
| kube_deployment_spec_replicas | replicas |
| kube_deployment_status_observed_generation | unless |
| kube_deployment_metadata_generation | unless |
| kube_pod_container_status_running | pods |
| kube_pod_container_status_restarts_total | restarts |
| kube_pod_container_resource_limits | memory/cpu cap |

## B.4. Container

| Metric | Use |
|--------|-----|
| container_memory_working_set_bytes | OOM risk |
| container_cpu_usage_seconds_total | CPU |
| container_cpu_cfs_throttled_seconds_total | throttling |

## B.5. PostgreSQL

| Metric | Use |
|--------|-----|
| pg_stat_activity_count | load |
| pg_settings_max_connections | load |
| pg_replication_lag_seconds | replica |
| pg_locks_count | blocking |

## B.6. pgBackRest

| Metric | Use |
|--------|-----|
| pgbackrest_backup_since_last_completion_seconds | stale backup |
| pgbackrest_wal_archive_status | WAL |
| pgbackrest_backup_error | failed backup |

## B.7. Kafka

| Metric | Use |
|--------|-----|
| kafka_consumergroup_lag | consumer |
| kafka_consumergroup_members | stalled |
| kafka_cluster_partition_under_replicated_partition | cluster |

## B.8. ClickHouse

| Metric | Use |
|--------|-----|
| clickhouse_readonly_replica | health |
| ClickHouseAsyncMetrics_ReplicasMaxAbsoluteDelay | replication |
| ClickHouseAsyncMetrics_ReplicasSumQueueSize | queue |
| ClickHouseProfileEvents_ReplicatedPartFailedFetches | fetch errors |

## B.9. Redis

| Metric | Use |
|--------|-----|
| redis_memory_used_bytes | memory |
| redis_memory_max_bytes | memory |
| redis_connected_clients | connections |
| redis_config_maxclients | connections |

## B.10. Application HTTP

Micrometer / custom: `*_requests_total`, `*_duration_seconds_bucket` — 5xx rate, latency quantiles.
