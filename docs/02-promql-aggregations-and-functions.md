# 02. PromQL: агрегации, rate, increase, histogram

## 2.1. Зачем агрегировать

На одном хосте несколько mount point, на Kafka — lag по partition, в K8s — несколько pod.  
Алерт «диск полон» обычно нужен **на хост** (worst mount) или **на кластер** (sum lag), а не 50 firing на каждую partition.

**Aggregation operators** схлопывают множество рядов:

```promql
sum by (instance) (...)
max by (instance) (...)
avg by (cluster, namespace) (...)
count by (job) (...)
min by (consumergroup) (...)
stddev by (instance) (...)
```

| Оператор | Типичный алерт |
|----------|----------------|
| `sum` | total lag, total errors |
| `max` | worst disk % на instance |
| `avg` | средняя CPU по ядрам → один % idle |
| `min` | осторожно: «лучший» partition скрывает проблему |
| `count` | число pod в состоянии |

### `by` vs `without`

```promql
sum without (cpu) (rate(node_cpu_seconds_total[5m]))
```

`without (cpu)` — суммировать, **убрав** label `cpu` из группировки.

```promql
sum by (instance) (rate(node_cpu_seconds_total{mode!="idle"}[5m]))
```

Явно указать, какие labels **оставить** в результате.

## 2.2. CPU node: от counter к процентам

`node_cpu_seconds_total` — **counter** с label `mode` (`idle`, `user`, `system`, …).

Idle fraction → busy %:

```promql
100 - (
  avg by (instance) (
    rate(node_cpu_seconds_total{
      service="order-hub-pgsql-cl1",
      job="node",
      instance=~"order-hub-pgsql-cl1-[0-9]+.*:9100",
      mode="idle"
    }[5m])
  ) * 100
) > 85
```

- `rate(...[5m])` — доля секунды CPU в секунду wall time.
- `avg by (instance)` — среднее по ядрам (если без mode фильтра — учитывайте все mode; здесь только idle).

## 2.3. RAM node (gauge)

```promql
(
  1 - node_memory_MemAvailable_bytes{
    service="order-hub-analytics-ch-cl1",
    job="node",
    instance=~"order-hub-analytics-ch-cl1-[0-9]+.*:9100"
  }
  / node_memory_MemTotal_bytes{
    service="order-hub-analytics-ch-cl1",
    job="node",
    instance=~"order-hub-analytics-ch-cl1-[0-9]+.*:9100"
  }
) * 100 > 90
```

`MemAvailable` предпочтительнее `Free` — учитывает cache reclaimable.

## 2.4. Disk usage (gauge через два gauge)

```promql
max by (instance) (
  (
    1 - node_filesystem_avail_bytes{
      service="order-hub-analytics-ch-cl1",
      environment="prod1",
      job="node",
      instance=~"order-hub-analytics-ch-cl1-[0-9]+.*:9100",
      fstype!~"^(fuse.*|tmpfs|cifs|nfs)"
    }
    / node_filesystem_size_bytes{
      service="order-hub-analytics-ch-cl1",
      environment="prod1",
      job="node",
      instance=~"order-hub-analytics-ch-cl1-[0-9]+.*:9100",
      fstype!~"^(fuse.*|tmpfs|cifs|nfs)"
    }
  ) * 100
) > 80
```

- `max by (instance)` — худший mount на машине.
- `fstype!~` — не алертить tmpfs/docker internal mounts.

## 2.5. rate, irate, increase

### rate

Средняя скорость прироста counter **за окно**, с extrapolation:

```promql
sum(rate(http_requests_total{status=~"5.."}[5m]))
```

Окно `[5m]` ≥ 4–5 scrape интервалов — стабильнее.

### irate

«Мгновенная» скорость по последним двум точкам — шумная, для алертов реже.

### increase

Прирост за окно (может быть дробным из-за extrapolation):

```promql
sum by (pod) (
  increase(kube_pod_container_status_restarts_total{
    namespace="order-hub-prod1",
    container="order-hub-api"
  }[15m])
) > 5
```

CrashLoop: >5 рестартов за 15m.

### Reset counter

При рестарте exporter counter падает — `rate()` обрабатывает reset; не используйте `delta()` на counter без понимания.

## 2.6. Окно `[...]` — как выбрать

| Scrape | Минимум окна | Рекомендация в алертах |
|--------|--------------|------------------------|
| 15s | ~1m | `[5m]` |
| 30s | ~2m | `[5m]` |
| 1m | ~5m | `[5m]` или `[10m]` |

Слишком короткое окно → дрожание алерта. Компенсируйте **`for:`**, не только `[1m]`.

## 2.7. Histogram и latency

Структура: `http_request_duration_seconds_bucket{le="0.5"}`, `_sum`, `_count`.

p99:

```promql
histogram_quantile(
  0.99,
  sum by (le, namespace) (
    rate(http_request_duration_seconds_bucket{namespace="order-hub-prod1"}[5m])
  )
) > 2
```

`le` **обязательно** в `sum by (le, ...)`.

## 2.8. topk / bottomk / quantile

```promql
topk(3, sum by (instance) (rate(errors_total[5m])))
```

Для алертов чаще фиксированный порог, не topk — topk полезен в дашбордах.

## 2.9. absent() и «метрика пропала»

```promql
absent(up{service="order-hub-pgsql-cl1", job="postgres"} == 1)
```

Редкий паттерн: алерт, если **нет** series (target удалён из мониторинга).  
Осторожно: `absent()` чувствителен к label set — matcher'ы должны **точно** совпадать.

Чаще достаточно `up{...} == 0`.

## 2.10. Примеры «перевод задачи → expr»

| Задача | Expr (скелет) |
|--------|----------------|
| Lag CG > 10k | `sum(kafka_consumergroup_lag{consumergroup="...", topic="..."}) > 10000` |
| Backup incr > 36h | `pgbackrest_backup_since_last_completion_seconds{backup_type="incr"} > 129600` |
| CH readonly | `clickhouse_readonly_replica{...} == 1` |
| 5xx > 5% RPS | `sum(rate(req{status=~"5.."}[5m])) / sum(rate(req[5m])) > 0.05` |

Секунды: 1h=3600, 36h=129600, 7d=604800, 7d+1h=608400.

---

Далее: логика `unless` и join — [03-promql-logic-and-alert-expressions.md](03-promql-logic-and-alert-expressions.md).
