# 10. Git: ветка, commit, Merge Request

Изменения в конфигурации алертов попадают в среду через Git: вы работаете в **своей ветке**, открываете **Merge Request** в основную ветку (обычно `master`), проходите **review у лида**, после merge всё остальное (проверки в CI, выкат конфигурации) выполняется **автоматически**. Ручной деплой от автора MR не требуется.

## 10.1. Подготовка

Обновите основную ветку и создайте ветку под задачу (имя — как принято у команды, например `TASK-123-clickhouse-alerts`):

```bash
git checkout master
git pull origin master
git checkout -b TASK-123-clickhouse-alerts
```

## 10.2. Правки и commit

Внесите изменения в нужные YAML-файлы (см. главу 05). Перед commit полезно посмотреть diff:

```bash
git status
git diff
```

Зафиксируйте изменения:

```bash
git add projects/order-hub/values-vmalert-vm.yaml
git commit -m "TASK-123: add ClickHouse infra alerts"
```

Сообщение commit — коротко **что** и **зачем**; номер задачи — если используете трекер.

## 10.3. Push и Merge Request

Отправьте ветку на remote:

```bash
git push -u origin TASK-123-clickhouse-alerts
```

В веб-интерфейсе Git hosting (GitLab и аналоги): **Create merge request** → целевая ветка **`master`**.  
Назначьте ревьюера (лид / ответственный за репозиторий). Кратко опишите суть в описании MR — по договорённости в команде.

## 10.4. Review и merge

Лид оставляет комментарии — вы правите код **в той же ветке**, делаете commit и снова `git push`. MR обновляется сам.

После approve лид (или вы, если есть права) нажимает **Merge**. Дальше pipeline и доставка конфигурации в кластер идут **без ваших действий**.

## 10.5. Конфликт с master

Если MR нельзя смержить — в `master` уже изменили те же файлы. Локально:

```bash
git fetch origin
git checkout TASK-123-clickhouse-alerts
git merge origin/master
```

Git пометит конфликтные участки маркерами `<<<<<<<`, `=======`, `>>>>>>>`. Откройте файл в редакторе:

- для **двух новых групп алертов** оставьте **обе** группы под `groups:` подряд;
- для **одной группы** объедините списки `- alert:` под одним `name:` и `interval:`;
- удалите все маркеры conflict.

```bash
git add <исправленные-файлы>
git commit -m "Resolve merge conflict with master"
git push
```

Если команда использует rebase вместо merge — действуйте по её правилам (`git rebase origin/master`, затем push).

---

Далее — отладка уже выкатанных правил: [11-debugging-alerts.md](11-debugging-alerts.md).
