# 12. Упражнения

Перед решением повторите главы 01–04. Ответы — в `<details>`.

---

## 12.1. Selector node exporter

Напишите selector `up` для всех node exporter нод кластера `order-hub-pgsql-cl1`, `prod1`, порт 9100.

<details><summary>Ответ</summary>

```promql
up{
  service="order-hub-pgsql-cl1",
  environment="prod1",
  job="node",
  instance=~"order-hub-pgsql-cl1-[0-9]+.*:9100"
} == 0
```

</details>

---

## 12.2. Disk percent

Почему алерт `node_filesystem_avail_bytes{...} > 8589934592` **не** эквивалентен «диск > 80%»?

<details><summary>Ответ</summary>

Сравниваются **байты** free space с константой, без учёта размера тома. На диске 100GB и 1TB одна и та же константа даёт разный смысл. Нужно `(1 - avail/size)*100`.

</details>

---

## 12.3. rate window

Scrape interval 30s. Почему `[1m]` для `rate()` в алерте — плохая идея?

<details><summary>Ответ</summary>

Мало точек в окне → нестабильный rate, ложные spikes. Рекомендация `[5m]` + `for`.

</details>

---

## 12.4. unless

В deployment rollout зачем `unless on(cluster, namespace, deployment)` а не голый `unless`?

<details><summary>Ответ</summary>

`on()` задаёт ключи join. Иначе лишние labels мешают match, suppression не срабатывает, алерт firing при каждом deploy.

</details>

---

## 12.5. YAML structure

Перечислите минимум 4 ошибки в [broken-rules-snippet.yaml](../examples/broken-rules-snippet.yaml).

<details><summary>Ответ</summary>

1. `- alert:` на уровне `groups`, вне `rules`.
2. PromQL: `job="node", ,`.
3. Trailing comma в `up{job="clickhouse",}`.
4. HAProxy вне блока `description`.
5. Имена алертов не PascalCase; `service: api-order-hub` для infra.
6. Postgres disk в том же diff, что CH (scope).

</details>

---

## 12.6. Kafka MR block

Задача: lag sum > 10000, `for` 15m, topic `orders-status-changed`, CG `kafka_consumer_order_hub_prod`, service `shared-events-kfk-cl1`, project order-hub.

Напишите группу целиком (name, interval, один alert).

<details><summary>Ответ</summary>

См. [examples/order-hub-values-vmalert-vm.yaml](../examples/order-hub-values-vmalert-vm.yaml) — `order-hub-kafka`.

</details>

---

## 12.7. pgBackRest

Объедините incr и diff в один alert `backup_type=~"incr|diff"` с порогом 129600. Что произойдёт на 8-й день после успешного diff?

<details><summary>Ответ</summary>

Diff backup раз в ~7 дней. Через 36h после diff метрика age > 129600 → **постоянный critical**, хотя incr может быть свежим.

</details>

---

## 12.8. Merge conflict

В master добавили группу `order-hub-kafka`. В вашей ветке — `order-hub-clickhouse-infra`. Сколько элементов `- name:` должно быть в `groups` после правильного merge?

<details><summary>Ответ</summary>

На **два больше**, чем было до обеих фич (обе группы сохраняются), если никто не удалял существующие группы.

</details>

---

## 12.9. Debugging

Explore: expr возвращает 2 series > порога. vmalert: 2 firing. Telegram: 1 сообщение. Это баг?

<details><summary>Ответ</summary>

Не обязательно. Alertmanager **group_by** (alertname, severity, project, …) склеивает несколько firing в одно уведомление. Проверьте текст — должны быть оба instance или worst case.

</details>

---

## 12.10. project label

Где задаётся `project=order-hub` на firing alert и почему нельзя продублировать в `labels:` правила?

<details><summary>Ответ</summary>

vmalert `external.label`. Дублирование вручную рискует рассинхроном с Alertmanager route и дублями matchers.

</details>
