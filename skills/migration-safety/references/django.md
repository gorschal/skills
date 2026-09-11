# Django migrations

Django — единственный источник правды по схеме и миграциям; другие инструменты
миграций в сервисах не используются.

## Смотреть реальный SQL

```bash
python manage.py makemigrations
python manage.py sqlmigrate app_label 0042          # SQL конкретной миграции
python manage.py showmigrations                     # что не применено
python manage.py makemigrations --check --dry-run   # CI: упасть, если модель без миграции
```

## `atomic = False`

По умолчанию миграция оборачивается в транзакцию. Отключать для:

- `CREATE INDEX CONCURRENTLY` (нельзя в транзакции);
- батч-backfill (нужны промежуточные commit).

```python
class Migration(migrations.Migration):
    atomic = False
    operations = [...]
```

Минус: при сбое миграция применится частично — операции делать идемпотентными и
допускающими повторный запуск.

## Индексы без блокировки

```python
from django.contrib.postgres.operations import AddIndexConcurrently

class Migration(migrations.Migration):
    atomic = False
    operations = [
        AddIndexConcurrently(
            "order",
            models.Index(fields=["user_id"], name="order_user_idx"),
        ),
    ]
```

`RemoveIndexConcurrently` — симметрично.

## Добавление полей

- **Новое `NOT NULL`-поле**: Django предложит one-off default и впишет `UPDATE`
  всех строк → на большой таблице CRITICAL. Безопасно: `null=True` → backfill
  батчами (отдельная `RunPython` с `atomic=False`) → отдельной миграцией снять
  `null` (паттерн `NOT VALID`/`VALIDATE`).
- На PG 11+ константный default не вызывает rewrite.

## `RunPython` — обратимость и транзакции

- Всегда `reverse_code`; если откат бессмыслен — `migrations.RunPython.noop`.
- Внутри — `apps.get_model("app", "Model")` (historical model), не прямой импорт.
- Backfill — батчи + `.iterator()`/срезы по PK, не `.all()`.

```python
def forwards(apps, schema_editor):
    Order = apps.get_model("shop", "Order")
    qs = Order.objects.filter(total__isnull=True)
    for order in qs.iterator(chunk_size=1000):
        order.total = order.amount
        order.save(update_fields=["total"])

operations = [migrations.RunPython(forwards, migrations.RunPython.noop)]
```

## Rename без поломки — `SeparateDatabaseAndState`

`RenameField`/`RenameModel` мгновенно ломают старый код при rolling-деплое:

```python
operations = [
    migrations.SeparateDatabaseAndState(
        state_operations=[migrations.RenameField("order", "amount", "total")],
        database_operations=[],  # БД не трогаем; колонку переключаем expand/contract
    )
]
```

## Прочее

- **Не редактировать применённые на проде миграции** — расхождение с
  `django_migrations`. Чистка — `squashmigrations` (после того, как старые
  применены везде).
- **Схему и данные разделять** на разные миграции (разные `atomic`, раздельный
  откат).
- **SQLite**: Django эмулирует сложный `ALTER` через пересоздание таблицы
  (table rebuild) — медленно и блокирует; на больших данных сигнал переезжать на
  Postgres.

## Антипаттерны

| ❌ | ✅ |
|---|---|
| `CREATE INDEX` без `CONCURRENTLY` | `AddIndexConcurrently` + `atomic=False` |
| `RunPython` без reverse | `reverse_code`/`noop` |
| Прямой импорт модели в `RunPython` | `apps.get_model` |
| Backfill `.all()` в транзакции | батчи + `atomic=False` |
| Схема и данные в одной миграции | раздельные |
| Редактирование применённой миграции | новая/`squashmigrations` |
| `RenameField` при zero-downtime | `SeparateDatabaseAndState` + expand/contract |

## Чек-лист

- [ ] `sqlmigrate` просмотрен.
- [ ] `makemigrations --check --dry-run` проходит в CI.
- [ ] Тяжёлые операции — `atomic=False` + `CONCURRENTLY`/батчи.
- [ ] `RunPython` с `reverse_code` и `apps.get_model`.
- [ ] Схема и данные — раздельно.
- [ ] Применённые миграции не редактировались.
