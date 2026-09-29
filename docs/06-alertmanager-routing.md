# 06. Alertmanager: маршрутизация и уведомления

vmalert **создаёт** alert; Alertmanager **маршрутизирует**, **группирует**, **дедуплицирует** и **шлёт** в Telegram, Mattermost, webhook.

## 6.1. Поток labels

```
expr → vmalert добавляет externalLabels (project=...)
     → AM получает alert с labels + annotations
     → route tree выбирает receiver
     → receiver → integration
```

Если **ни один** route не matched — default receiver (в типичном config — **`drop`**, алерт теряется для chat).

## 6.2. Route tree (концепция)

```yaml
route:
  receiver: drop
  group_by: [alertname, severity, source, project]
  group_wait: 30s
  repeat_interval: 4h
  routes:
    - receiver: keephq
      continue: true
    - matchers:
        - project="order-hub"
      receiver: order-hub-all
```

| Параметр | Смысл |
|----------|--------|
| `group_by` | склеить несколько firing в одно сообщение |
| `group_wait` | подождать N, собрать batch |
| `repeat_interval` | не спамить чаще |
| `continue: true` | после match **продолжить** искать другие routes (KeepHQ + project chat) |
| `matchers` | label matchers Prometheus 2.x style |

## 6.3. Matchers

```yaml
matchers:
  - project="order-hub"
```

Regex:

```yaml
matchers:
  - severity=~"warning|critical"
```

**project** на алерте должен совпадать с `external.label` vmalert этого project.

## 6.4. Receivers

```yaml
receivers:
  - name: drop
  - name: order-hub-all
    telegram_configs:
      - bot_token: ""      # из CI variables / sealed secrets
        chat_id: 0
        parse_mode: HTML
        send_resolved: true
    mattermost_configs:
      - webhook_url: ""
        send_resolved: true
        username: "Order Hub Alertmanager"
```

Шаблоны `message` / `text` часто копируют из существующего receiver и меняют username.

## 6.5. severity и inhibit (концепция)

| severity | Использование |
|----------|---------------|
| warning | деградация |
| critical | outage |

**Inhibit rules** (если настроены): critical подавляет warning с тем же `alertname`/`instance` — зависит от org config.

## 6.6. Почему алерт «есть в vmalert», но нет в Telegram

| Причина | Проверка |
|---------|----------|
| project label ≠ route | AM UI → labels |
| receiver `drop` | default route |
| Ошибка Telegram API | AM logs `Bad Request`, HTML parse |
| group_wait ещё не истёк | подождать |
| Silence | AM / KeepHQ silence |

Telegram и Mattermost — **независимые** интеграции: одна может работать, другая — нет.

## 6.7. Resolved notifications

`send_resolved: true` — при expr false приходит « восстановлено ».  
Если firing **не дошёл** до Telegram (ошибка send), resolved может **не отправиться** (отдельный notification log per integration).

## 6.8. KeepHQ / webhook

Отдельный route с `continue: true` шлёт **копию** всех (или почти всех) alerts в ITSM.  
Не удаляйте `continue: true` у project routes без понимания — иначе project channel не получит alert.

---

Далее: K8s — [07-kubernetes-alerts.md](07-kubernetes-alerts.md).
