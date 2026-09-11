# Миграции Django

Базовые правила безопасных миграций. Zero-downtime, concurrent-индексы,
батч-backfill, `SeparateDatabaseAndState` — в навыке `migration-safety`.

## Смотреть реальный SQL

Django скрывает SQL за операциями — перед выкаткой смотри, что уйдёт в БД:

```bash
python manage.py makemigrations
python manage.py sqlmigrate app_label 0042          # SQL конкретной миграции
python manage.py showmigrations                     # что не применено
python manage.py makemigrations --check --dry-run   # CI: упасть, если модель без миграции
```

## Основные правила

1. **Изменение данных — только в `RunPython`.** Никогда вручную через shell на проде.
2. **`RunPython` всегда с `reverse_code`** (для необратимого — `RunPython.noop`).
3. **В `RunPython` — `apps.get_model(...)`**, а не прямой импорт модели: прямой
   импорт сломается при будущих изменениях модели.
4. **Схему и данные разделять** на разные миграции (разный риск, раздельный откат).
5. **Применённые на проде миграции не редактировать** — расхождение с
   `django_migrations`. Для чистки — `squashmigrations`.
6. **Проверять обратную совместимость** при rolling-деплое.

## `RunPython`

```python
from django.db import migrations

def forwards(apps, schema_editor):
    Order = apps.get_model("shop", "Order")
    # backfill батчами по PK, а не один UPDATE на всю таблицу
    qs = Order.objects.filter(total__isnull=True)
    for order in qs.iterator(chunk_size=1000):
        order.total = order.amount
        order.save(update_fields=["total"])

def backwards(apps, schema_editor):
    pass

class Migration(migrations.Migration):
    dependencies = [("shop", "0003_...")]
    operations = [migrations.RunPython(forwards, backwards)]
```

Для тяжёлого backfill: `atomic = False` на миграции + батчи + `iterator()`.
Подробности — в навыке `migration-safety`.

## Добавление NOT NULL поля

Django предложит one-off default и впишет его `UPDATE`-ом всех строк — на большой
таблице это опасно. Безопасная последовательность:

1. Добавить поле с `null=True` (миграция схемы).
2. Backfill данных отдельной `RunPython`-миграцией (батчами, `atomic=False`).
3. Отдельной миграцией снять `null`.

## Опасные операции

| Операция | Риск |
|---|---|
| Удаление поля/таблицы | потеря данных; удалять только после отказа кода от поля |
| Добавление `NOT NULL` с default | блокирующий `UPDATE` всех строк |
| `RenameField`/`RenameModel` | ломает старый код при rolling-деплое |
| Обычный `AddIndex` | блокирует таблицу (нужен concurrent) |
| Изменение типа поля | перезапись таблицы |

## Чек-лист

- [ ] SQL проверен через `sqlmigrate`.
- [ ] `makemigrations --check --dry-run` проходит в CI.
- [ ] Данные изменяются только в `RunPython`, с `reverse_code`.
- [ ] В `RunPython` — `apps.get_model`.
- [ ] Схема и данные — в разных миграциях.
- [ ] Применённые миграции не редактировались.
- [ ] Опасные операции (NOT NULL, rename, индексы) отработаны безопасно.
- [ ] Backfill — батчами, с `atomic=False` (см. `migration-safety`).
