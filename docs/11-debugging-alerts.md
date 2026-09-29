# 11. Отладка алертов: от Explore до уведомления

Диагностика **строго по цепочке**. Пропуск шага ведёт к ложным выводам («Telegram сломан», хотя expr не firing).

```
Explore (метрика есть?)
    → expr как в YAML (порог достижим?)
        → vmalert /api/v1/rules (группа загружена? parse ok?)
            → vmalert /api/v1/alerts (pending/firing?)
                → Alertmanager (receiver? silences?)
                    → Telegram / Mattermost (integration logs)
```

Деплой после merge **автоматический** — если правило не появилось сразу, подождите завершения pipeline на master или уточните у platform SRE типичную задержку (минуты).

---

## 11.1. Grafana Explore — разведка

### 1.1. Выбор datasource

Откройте `projects/<name>/values-vmalert-vm.yaml` → `server.datasource.url`.  
В Grafana выберите datasource, который указывает на **тот же** vmselect/thanos (имя в UI может отличаться от URL — спросите коллег или смотрите datasource settings).

### 1.2. Поиск metric name

```promql
{service="order-hub-analytics-ch-cl1", environment="prod1"}
```

Или:

```promql
{__name__=~"ClickHouse.*", service="order-hub-analytics-ch-cl1"}
```

### 1.3. Проверка up

```promql
up{
  service="order-hub-analytics-ch-cl1",
  job="clickhouse",
  instance=~"order-hub-analytics-ch-cl1-[0-9]+.*:9116"
}
```

Значения 1/0, не empty.

### 1.4. Полный expr

Скопируйте **без изменений** из `expr: >-`. Query type: **Instant** (для алертов).

### 11.5. Исторический range

Переключите graph на 24h / 7d. Если проблема была «вчera», увидите пики.  
Если expr **никогда** не пересекал порог — алерт не firing **корректно** (или порог неверен).

### 11.6. Table / Inspect

Посмотрите **все labels** series — скопируйте в YAML matcher'ы буква в букву.

---

## 11.2. Типичные ошибки Explore

| Симптом | Причина | Fix |
|---------|---------|-----|
| No data | wrong datasource | другой values file |
| No data | `environment="prod"` vs `prod1` | исправить label |
| Empty instant | порог слишком высок | временно убрать `> N` |
| Too many series | широкий selector | сузить job, instance=~ |
| Parse error | `, ,` в expr | YAML/expr edit |

---

## 11.3. vmalert после выката

### Port-forward

```bash
kubectl port-forward -n external-monitoring \
  svc/vmalert-order-hub-vm-victoria-metrics-alert-server 8880:8880
```

Имя service = `<helm-release>-victoria-metrics-alert-server`.

### Endpoints

| Path | Что смотреть |
|------|----------------|
| `/` | UI, список groups |
| `/api/v1/rules` | JSON: `groups[].name`, `rules[].name`, `state` |
| `/api/v1/alerts` | active: pending/firing, labels |

```bash
curl -s http://localhost:8880/api/v1/rules | jq '.data.groups[] | select(.name|test("clickhouse"))'
curl -s http://localhost:8880/api/v1/alerts | jq '.data.alerts[] | select(.labels.alertname|test("Clickhouse"))'
```

### Состояния

| state | Интерпретация |
|-------|----------------|
| inactive | условие false |
| pending | true, ждёт `for` |
| firing | отправлено в AM |
| err (в rules) | expr invalid — **не работает** |

### Группы нет в API

- MR не merged / pipeline master failed.
- Правили не тот project file.
- Helm release другого project (опечатка folder name vs external.label).

---

## 11.4. Alertmanager

```bash
kubectl port-forward -n external-monitoring svc/alertmanager 9093:9093
curl -s http://localhost:9093/api/v2/alerts | jq '.[] | {alertname: .labels.alertname, project: .labels.project, receiver: .receivers[0].name}'
```

### project label

Должен совпадать с route:

```yaml
# vmalert
external.label: "project=order-hub"

# alertmanager
matchers:
  - project="order-hub"
```

Опечатка `order_hub` → receiver **drop**.

### Silences / inhibition

UI AM → Silences. Maintenance мог заглушить `alertname` или `service`.

---

## 11.5. Уведомления

### Mattermost пришёл, Telegram нет

Разные `*\_configs` в receiver. Логи AM:

```bash
kubectl logs -n external-monitoring alertmanager-0 --tail=100 | grep -i telegram
```

Часто: HTML entity, неверный `chat_id`, revoked bot token.

### group_wait

Первое сообщение может задержаться на `group_wait` (например 30s) — не паниковать секунду в секунду.

### repeat_interval

Повтор того же firing — не чаще `repeat_interval` (например 4h).

---

## 11.6. Ложные firing

| Паттерн | Fix |
|---------|-----|
| Deploy K8s | unless generation |
| Weekly diff backup | отдельный alert 608400 |
| Kafka partition noise | sum lag, не min |
| tmpfs disk 100% | fstype filter |
| Scrape blip | увеличить `for` |

---

## 11.7. Пропущенный инцидент (алерт не firing)

1. Метрики пропали раньше порога → добавить `up` exporter.
2. Expr слишком строгий → сравнить labels с `/metrics` на хосте.
3. vmalert down → meta-alert (обычно platform).
4. Алерт в **thanos** file, метрика только в **vm** (или наоборот).

---

## 11.8. Локальный helm lint (как CI)

```bash
helm repo add vm https://victoriametrics.github.io/helm-charts/
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add keephq https://keephq.github.io/helm-charts
helm repo update
helm pull vm/victoria-metrics-alert --version 0.49.0 --untar --untardir /tmp/charts
helm pull prometheus-community/alertmanager --version 2.0.0 --untar --untardir /tmp/charts
helm lint /tmp/charts/victoria-metrics-alert -f projects/order-hub/values-vmalert-vm.yaml
```

---

## 11.9. Тест доставки (осторожно)

В **личной ветке** (не merge в master без удаления):

```yaml
expr: vector(1) > 0
for: 1m
alert: OrderHubDeliveryTestRemoveMe
```

Убедиться, что AM route доходит до чата → удалить правило.

---

## 11.10. Сводная таблица

| Explore | rules API | alerts API | AM | Chat |
|---------|-----------|------------|-----|------|
| No data | — | — | — | Fix expr/datasource |
| OK | group missing | — | — | deploy/file/release |
| OK | parse err | — | — | Fix PromQL |
| OK | ok | inactive | — | OK or threshold |
| OK | ok | pending | — | Wait `for` |
| OK | ok | firing | empty | vmalert→AM |
| OK | ok | firing | drop | project route |
| OK | ok | firing | ok | integration |

---

[12-exercises.md](12-exercises.md)
