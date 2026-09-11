---
name: migration-safety
description: >
  Use before applying Django migrations to production: will the migration lock
  the table, break old code during rolling deploy, lose data or be irreversible.
  Covers Postgres locks, zero-downtime expand/contract, CREATE INDEX CONCURRENTLY,
  NOT VALID/VALIDATE, batched backfill, RunPython, atomic=False, lock_timeout.
  Триггеры: Django migration, makemigrations, migrate, RunPython, AddIndexConcurrently,
  zero-downtime, lock_timeout, CONCURRENTLY, backfill, NOT VALID, блокировка таблицы.
  Подобрать индекс под запрос — навык postgres.
license: MIT
compatibility: opencode
metadata:
  version: "1.0.0"
  domain: database
  triggers: Django migration, makemigrations, migrate, RunPython, zero-downtime, CONCURRENTLY, backfill, lock_timeout, NOT VALID
  role: specialist
  scope: review
  output-format: report
  related-skills: django, postgres, python
---

# Migration Safety

Безопасность миграций **Django** перед прод-деплоем. Django — единственный
источник правды по схеме и миграциям: Alembic и `create_all` в других сервисах
не используются.

Цель — поймать операции, которые блокируют таблицу под нагрузкой, ломают старый
код при rolling-деплое, теряют данные или необратимы.

## Когда применять

- Перед применением миграций на проде; при ревью PR с новой миграцией.
- Добавление колонки/индекса/constraint/FK на непустой таблице.
- Написание data-миграций и backfill.

## Контекст (установить первым)

1. **СУБД и версия** — Postgres (11/12+ меняют безопасность операций) или SQLite.
2. **Модель деплоя** — короткий downtime или zero-downtime/rolling.
3. **Размер таблиц** — операция, безопасная на 1k строк, кладёт прод на 50M.
4. **Объём/нагрузка** — есть ли активный трафик к таблице.

## Уровни риска

| Уровень | Что |
|---|---|
| **CRITICAL** | гарантированный простой/потеря данных: table rewrite, долгий `ACCESS EXCLUSIVE` на большой таблице, `DROP COLUMN` с данными, backfill всей таблицы одной транзакцией |
| **HIGH** | блокировка записи или поломка старого кода: индекс без `CONCURRENTLY`, `RENAME`, новый `NOT NULL`/`UNIQUE`/FK без двухфазного приёма |
| **MEDIUM** | нет `lock_timeout`, нет обратной операции (`RunPython` без reverse), схема и данные в одной миграции |
| **LOW** | стиль, именование |

Не раздувать риск: на пустой/маленькой таблице `CREATE INDEX` без `CONCURRENTLY` —
LOW, а не HIGH. Уровень привязывать к объёму и модели деплоя.

## Быстрый чек-лист

- [ ] `ADD COLUMN` с volatile/вычисляемым default или новый `NOT NULL` на непустой
  таблице → rewrite.
- [ ] `CREATE INDEX` без `CONCURRENTLY` → блокирует запись.
- [ ] `ALTER COLUMN TYPE` → почти всегда rewrite + `ACCESS EXCLUSIVE`.
- [ ] FK/`UNIQUE`/`NOT NULL` одним шагом → `NOT VALID` → `VALIDATE`.
- [ ] `RENAME` колонки/таблицы → ломает старый код (zero-downtime — expand/contract).
- [ ] `DROP COLUMN`/`DROP TABLE` → потеря данных; только в contract-фазе.
- [ ] Backfill всей таблицы одной транзакцией → длинные локи, WAL, лаг реплик.
- [ ] Нет `lock_timeout` перед DDL → очередь локов ко всей таблице.
- [ ] Нет обратной операции (`reverse_code`) → миграцию нельзя откатить.

## Процесс

1. Собрать миграции на ревью: неприменённые файлы в `*/migrations/`,
   `python manage.py makemigrations --check --dry-run`.
2. Посмотреть реальный SQL: `python manage.py sqlmigrate app NNNN`.
3. Прогнать каждую операцию по чек-листу и справочнику.
4. Классифицировать риск и объяснить, почему опасно именно при вашей модели деплоя.
5. Предложить безопасную альтернативу (expand/contract, `CONCURRENTLY`, батчи, таймауты).
6. Отчёт по [references/report.md](references/report.md); правки — по согласованию.

## Django-инструменты

```bash
python manage.py makemigrations
python manage.py sqlmigrate app_label 0042        # реальный SQL
python manage.py showmigrations                    # что не применено
python manage.py makemigrations --check --dry-run  # CI
```

- Данные — только в `RunPython`, всегда с `reverse_code` (`RunPython.noop`, если
  откат бессмысленен).
- В `RunPython` — `apps.get_model(...)`, не прямой импорт модели.
- Схема и данные — в разных миграциях.
- Применённые миграции не редактировать; чистка — `squashmigrations`.
- `atomic = False` для `CREATE INDEX CONCURRENTLY` и батч-backfill.
- `AddIndexConcurrently`/`RemoveIndexConcurrently` из `django.contrib.postgres`.

Подробно: [references/django.md](references/django.md).

## Postgres-операции

| Операция | Риск | Безопасно |
|---|---|---|
| `ADD COLUMN` без default | LOW | да |
| `ADD COLUMN` с константным default | LOW (PG11+) | да |
| `ADD COLUMN` с volatile default (`now()`, `uuid()`) | CRITICAL | nullable → backfill → default отдельно |
| `ADD COLUMN ... NOT NULL` на непустой | CRITICAL | nullable → backfill → `NOT NULL` |
| `ALTER COLUMN TYPE` | CRITICAL | новая колонка → backfill → переключение |
| `CREATE INDEX` | HIGH | `CONCURRENTLY` (вне транзакции) |
| `ADD FOREIGN KEY` | HIGH | `NOT VALID` → `VALIDATE` |
| `ADD UNIQUE` | HIGH | `CREATE UNIQUE INDEX CONCURRENTLY` → `USING INDEX` |
| `RENAME` | HIGH | expand/contract |
| `DROP COLUMN`/`TABLE` | HIGH/CRITICAL | только в contract-фазе |
| Backfill всей таблицы | CRITICAL | батчами по PK с commit между |

- DDL берёт `ACCESS EXCLUSIVE` (блокирует всё, включая чтение) и/или переписывает
  таблицу. Очередь локов: даже мгновенный лок ждёт текущие запросы и блокирует
  новые.
- Перед DDL: `SET lock_timeout = '3s'`.

Подробно: [references/operations.md](references/operations.md).

## Модели деплоя

- **Короткий downtime** (стоп → миграция → старт): обратная совместимость не
  критична, главное — длительность блокировки.
- **Zero-downtime/rolling**: схема совместима и со старым, и с новым кодом →
  **expand/contract** (добавить → двойная запись → backfill → переключить →
  удалить).

При сомнении оценивать по более строгой (zero-downtime) модели.

Подробно: [references/deploy-models.md](references/deploy-models.md).

## Запрещённые паттерны

| ❌ Запрещено | ✅ Правильно |
|---|---|
| `CREATE INDEX` на живой таблице | `CONCURRENTLY` + `atomic = False` |
| `NOT NULL`/FK/`UNIQUE` одним шагом | `NOT VALID` → `VALIDATE` |
| Backfill всей таблицы в транзакции | батчи по PK, вне транзакции |
| `RENAME` при zero-downtime | expand/contract |
| `DROP COLUMN` до отказа кода | contract-фаза после выката |
| DDL без `lock_timeout` | `SET lock_timeout` |
| `RunPython` без reverse | `reverse_code`/`noop` |
| Схема и данные в одной миграции | раздельные миграции |
| Редактирование применённой миграции | новая миграция/`squashmigrations` |
| Прямой импорт модели в `RunPython` | `apps.get_model` |

## Чек-лист перед применением

- [ ] `lock_timeout` выставлен перед тяжёлым DDL.
- [ ] Тяжёлые операции — `CONCURRENTLY`/батчи/`NOT VALID`→`VALIDATE`.
- [ ] Backfill вынесен из транзакции и идемпотентен.
- [ ] Все операции обратимы (или необратимость осознана).
- [ ] Схема совместима со старым кодом (zero-downtime).
- [ ] Есть свежий бэкап перед деструктивными операциями.
- [ ] Оценено на объёме прода, а не dev-базы.
- [ ] `sqlmigrate` проверен; `makemigrations --check --dry-run` проходит.

## Справочники

| Тема | Reference | Загружать когда |
|---|---|---|
| Postgres-операции и блокировки | [references/operations.md](references/operations.md) | Таблица «операция → риск → альтернатива» |
| Django-специфика | [references/django.md](references/django.md) | `atomic=False`, `AddIndexConcurrently`, `RunPython` |
| Модели деплоя | [references/deploy-models.md](references/deploy-models.md) | downtime vs zero-downtime, expand/contract |
| Отчёт аудита | [references/report.md](references/report.md) | Формат вывода и правки |

## Связанные навыки

- `django` — модели, ORM, миграции.
- `postgres` — подбор индекса под запрос, блокировки, планы.
- `python-audit` — общий аудит готовности проекта.
