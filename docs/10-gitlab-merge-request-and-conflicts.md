# 10. GitLab: ветка, Merge Request, CI, конфликты

В репозитории конфигурации алертинга (например **external-monitoring**) изменения попадают в prod через **Git**: ветка → MR → review → merge в `master`.  
**Validate** (helm lint) запускается на MR и на master. **Деплой** после merge автоматизирован — в обычном процессе не требуется ручной `helm upgrade` от автора MR.

## 10.1. Предварительные условия

- Доступ к GitLab репозиторию (clone по HTTPS/SSH).
- Права на создание веток и MR.
- Локально: git, опционально helm (lint как CI).

```bash
git clone git@gitlab.example.com:infra/external-monitoring.git
cd external-monitoring
git checkout master
git pull origin master
```

## 10.2. Ветка под задачу

Имя ветки — по соглашению команды, часто:

```text
MNTR-123_clickhouse-alerts
feature/order-hub-kafka-lag
```

```bash
git checkout -b MNTR-123_clickhouse-alerts
```

Одна ветка = одна логическая задача (или связанный набор правил одного project).

## 10.3. Правки только нужных файлов

| Задача | Файлы |
|--------|--------|
| Алерт в существующем project | `projects/<name>/values-vmalert-vm.yaml` и/или `-thanos.yaml` |
| Новый project | новая папка + `shared/values-alertmanager.yaml` |
| Новый канал / route | `shared/values-alertmanager.yaml` |

Перед commit:

```bash
git status
git diff
```

Убедитесь, что нет случайных правок и секретов.

## 10.4. Commit и push

```bash
git add projects/order-hub/values-vmalert-vm.yaml
git commit -m "MNTR-123: ClickHouse infra alerts for order-hub"
git push -u origin MNTR-123_clickhouse-alerts
```

## 10.5. Создание MR

GitLab → **Create merge request** → target **`master`**.

Шаблон описания:

```markdown
## Задача
MNTR-123 — ClickHouse order-hub-analytics-ch-cl1, prod1, health + disk

## Datasource
vmselect (проверено в Explore, скрин / имя datasource: ...)

## Изменения
- projects/order-hub/values-vmalert-vm.yaml
- группа: order-hub-clickhouse-infra
- алерты: OrderHubClickhouseExporterDown, ... (полный список)

## Out of scope
- PostgreSQL, Kafka — не трогались

## Checklist
- [x] helm lint локально / CI
- [x] expr без синтаксических ошибок
```

Assignee / reviewer — по правилам команды.

## 10.6. CI pipeline (validate)

Stage **validate** в `.gitlab-ci.yml`:

1. helm + pull charts (alertmanager, keep, victoria-metrics-alert).
2. `helm lint` на shared values.
3. Цикл `helm lint` по **каждому** `projects/*/values-vmalert-*.yaml`.

Pipeline на **merge_request_event** и на **master**.

CI **не** проверяет PromQL на live metrics.

После merge конфигурация применяется **автоматически** (org-specific pipeline на `master`).

Ручной деплой, kubeconfig и helm upgrade **не входят** в стандартный цикл автора MR: ваша ответственность — корректный YAML, зелёный **validate**, пройденный review.  
Проверку «алерт появился в vmalert» выполняют после выката по главе 11 (если есть доступ к кластеру).

## 10.7. Review и merge

Комментарии → commits в ветку → push.  
Merge после approve и зелёного pipeline.

## 10.8. Если pipeline красный не из‑за вашего файла

Lint проходит по **всем** `projects/*/values-vmalert-*.yaml`.  
Если упал чужой project — варианты: rebase на master (fix уже merged), сообщить maintainer, **не** «чинить» чужой project без задачи.

Если упал **ваш** файл — читайте вывод `helm lint`: номер строки, `Error:`, YAML indentation.

## 10.9. Конфликты с master

```bash
git fetch origin
git checkout MNTR-123_clickhouse-alerts
git merge origin/master
```

Маркеры в файле:

```yaml
<<<<<<< HEAD
... ваша версия ...
=======
... master ...
>>>>>>> origin/master
```

**Обе группы алертов** (если обе нужны) — оставить подряд в `groups:`:

```yaml
        - name: order-hub-kafka
          rules: [...]
        - name: order-hub-clickhouse-infra
          rules: [...]
```

Конфликт **внутри одной группы** — объединить списки `- alert:` под одним `name` / `interval`.

`shared/values-alertmanager.yaml`: сохранить **оба** route и **оба** receiver.

```bash
git add <files>
git commit -m "Resolve merge conflict with master"
git push
```

Не оставлять conflict markers. Не удалять чужие rules без согласования.

### Пример: оба добавили группу в конец `groups`

Master добавил kafka-группу, вы — clickhouse. Итог в файле:

```yaml
      groups:
        # ... существующие группы без изменений ...
        - name: order-hub-kafka
          interval: 30s
          rules:
            - alert: OrderHubKafkaConsumerLagHigh
              ...
        - name: order-hub-clickhouse-infra
          interval: 30s
          rules:
            - alert: OrderHubClickhouseExporterDown
              ...
```

Проверьте отступ `groups:` — он должен совпадать с остальным файлом (часто 6 пробелов до `- name:`).

### Пример: конфликт в одной группе

Два разработчика добавили **разные** `- alert:` в `order-hub-postgres-infra`.  
Объедините под одним `name:` / `interval:`:

```yaml
        - name: order-hub-postgres-infra
          interval: 30s
          rules:
            - alert: OrderHubPostgresExporterDown
              ...
            - alert: OrderHubPostgresDiskUsageCritical
              ...
            - alert: OrderHubPostgresNewAlertFromMaster
              ...
            - alert: OrderHubPostgresNewAlertFromBranch
              ...
```

Порядок алертов внутри `rules` не важен для vmalert.

## 10.10. Rebase (опционально)

```bash
git rebase origin/master
git push --force-with-lease
```

Только если так принято в команде.

## 10.11. Чеклист MR

- [ ] Актуальный master в ветке (merge/rebase).
- [ ] Explore проверен на datasource из values.
- [ ] groups/rules структура корректна.
- [ ] Pipeline validate зелёный.
- [ ] Конфликты разрешены.

---

Далее: [11-debugging-alerts.md](11-debugging-alerts.md).
