# PromQL и конфигурация алертов — полное руководство

Самодостаточная документация: от базовых понятий мониторинга до PromQL, YAML-правил vmalert, маршрутизации Alertmanager и практики работы в Git-репозитории с автоматической валидацией и деплоем.

Примеры используют **вымышленную** компанию Acme и проект **order-hub** (без реальных URL, токенов и имён прод-систем).

## Для кого и что внутри

Материал рассчитан на инженера, который:

- впервые пишет PromQL для алертов;
- правит Helm values с группами правил;
- открывает Merge Request в репозиторий формата «один проект — один vmalert — общий Alertmanager».

Читать **по номерам глав** — каждая следующая опирается на предыдущую.

## Оглавление

| Глава | Файл | Содержание |
|-------|------|------------|
| 00 | [docs/00-observability-from-zero.md](docs/00-observability-from-zero.md) | Метрики, Prometheus, scrape, типы данных, жизненный цикл алерта |
| 01 | [docs/01-promql-selectors-and-types.md](docs/01-promql-selectors-and-types.md) | PromQL: модель данных, selectors, литералы, простые выражения |
| 02 | [docs/02-promql-aggregations-and-functions.md](docs/02-promql-aggregations-and-functions.md) | Агрегации, `rate`/`increase`, математика, histogram |
| 03 | [docs/03-promql-logic-and-alert-expressions.md](docs/03-promql-logic-and-alert-expressions.md) | `and`/`or`/`unless`, `on`/`ignoring`, типовые expr для алертов |
| 04 | [docs/04-yaml-from-zero-to-alert-rules.md](docs/04-yaml-from-zero-to-alert-rules.md) | YAML с нуля, структура groups/rules, аннотации |
| 05 | [docs/05-vmalert-values-and-repository-layout.md](docs/05-vmalert-values-and-repository-layout.md) | Datasource, external.label, layout `projects/` и `shared/` |
| 06 | [docs/06-alertmanager-routing.md](docs/06-alertmanager-routing.md) | Routes, receivers, grouping, severity |
| 07 | [docs/07-kubernetes-alerts.md](docs/07-kubernetes-alerts.md) | kube-state, cAdvisor, rollout, ресурсы |
| 08 | [docs/08-infra-metrics-pg-kafka-ch-redis.md](docs/08-infra-metrics-pg-kafka-ch-redis.md) | PostgreSQL, pgBackRest, Kafka, ClickHouse, Redis |
| 09 | [docs/09-designing-alert-coverage.md](docs/09-designing-alert-coverage.md) | Как закрыть задачу мониторинга без лишнего scope |
| 10 | [docs/10-gitlab-merge-request-and-conflicts.md](docs/10-gitlab-merge-request-and-conflicts.md) | Ветка, MR, CI `helm lint`, разрешение конфликтов |
| 11 | [docs/11-debugging-alerts.md](docs/11-debugging-alerts.md) | Explore → vmalert → Alertmanager → уведомления |
| 12 | [docs/12-exercises.md](docs/12-exercises.md) | Задачи с разбором |
| A | [docs/appendix-a-labels-and-naming.md](docs/appendix-a-labels-and-naming.md) | Соглашения по labels, портам, именам алертов |
| B | [docs/appendix-b-metrics-catalog.md](docs/appendix-b-metrics-catalog.md) | Справочник частых метрик |

## Примеры в репозитории

- [examples/order-hub-values-vmalert-vm.yaml](examples/order-hub-values-vmalert-vm.yaml) — рабочий каркас групп (K8s, PG, Kafka, ClickHouse).
- [examples/broken-rules-snippet.yaml](examples/broken-rules-snippet.yaml) — антипримеры для самопроверки.

## Быстрый путь «первая задача»

1. Глава 00–03 — понимание PromQL.  
2. Глава 04–05 — куда класть правило в YAML.  
3. Grafana Explore — проверить expr на нужном datasource.  
4. Глава 10 — MR; дождаться зелёного pipeline.  
5. Глава 11 — после merge убедиться, что правило в vmalert и маршрут не `drop`.
