# Appendix A. Labels, порты, именование

## A.1. Порты в label instance

| job | Port | Метрики |
|-----|------|---------|
| node | 9100 | filesystem, cpu, memory |
| postgres | 9187 | pg_exporter |
| pgbouncer | 9127 | pool stats |
| pgbackrest | 9854 | backup |
| clickhouse | 9116 | CH + up |

Kafka — порт зависит от установки; смотреть `up{job="kafka",...}` в Explore.

## A.2. Regex instance

```promql
instance=~"order-hub-analytics-ch-cl1-[0-9]+.*:9116"
```

- `[0-9]+` — номер ноды в имени хоста.
- `.*` — доменный суффикс.
- Escaping в YAML не нужен при `>-`.

## A.3. environment

| Значение | Смысл |
|----------|--------|
| prod1 | production |
| stg1 | staging |

Заявка «prod» часто мапится на label `prod1`.

## A.4. Имена алертов

`OrderHub` + `Component` + `Problem`, PascalCase.

Примеры: `OrderHubClickhouseExporterDown`, `OrderHubPgbackrestIncrementalBackupStale`.

## A.5. Label service

| Контекст | service |
|----------|---------|
| PG cluster | order-hub-pgsql-cl1 |
| CH cluster | order-hub-analytics-ch-cl1 |
| K8s API | order-hub-api |
| Kafka (logical) | order-hub-kafka |

## A.6. source

`vmselect` — файл `-vm.yaml`; `thanos` — `-thanos.yaml`.

## A.7. project

Только через `external.label`. Route AM: `project="<folder-name>"`.

## A.8. Пороги времени (seconds)

| Human | sec |
|-------|-----|
| 1h | 3600 |
| 36h | 129600 |
| 7d | 604800 |
| 7d+1h | 608400 |
