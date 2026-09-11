# Django-линза

Глубокий разбор Django-проекта по линзам. Грузить только нужную. Префиксы находок:
`ARC`, `SEC`, `CLN`, `LEG`, `DEP`, `TST`, `TSK`.

## Линзы

| Линза          | Что проверяет                                                  |
| -------------- | -------------------------------------------------------------- |
| `architecture` | структура, модели, views, URLs, формы, сервисы, signals, admin |
| `security`     | OWASP, SQLi, XSS, CSRF, секреты, access control, settings      |
| `cleanup`      | мёртвый код, неиспользуемые импорты, TODO/FIXME, дубли         |
| `legacy`       | deprecated зависимости/импорты, незавершённые рефакторинги     |
| `deploy`       | prod-settings, security headers, БД, Docker, CI, healthcheck   |
| `tests`        | тесты без assertions, моки без проверок, хрупкие тесты         |
| `tasks`        | Django Tasks: идемпотентность, ошибки, повторы                 |

## architecture (`ARC`)

- Стандартная структура; приложения по доменам, не «god app».
- Модели: явные `on_delete`/`related_name`, индексы, `Meta`, нет бизнес-логики.
- Views тонкие, логика в сервисах; CBV/FBV уместны.
- URLs: `path()`, `reverse()`, namespaces.
- Формы: валидация в `clean_*`, не во view.
- Services/selectors: DI, без `request`.
- Signals — только для decoupling, зарегистрированы в `apps.ready()`.
- Admin: `list_display`/`search_fields`, нет чувствительных данных без защиты.

## security (`SEC`)

- `DEBUG=False`, `ALLOWED_HOSTS` ограничен, `SECRET_KEY` из env.
- HTTPS: `SECURE_SSL_REDIRECT`, secure cookies, HSTS, `X_FRAME_OPTIONS`.
- CSRF: нет `@csrf_exempt` без причины.
- XSS: нет `|safe`/`mark_safe` на вводе.
- SQL: только ORM/параметры.
- IDOR: `get_queryset()` фильтрует по пользователю.
- `fields` без `"__all__"` (mass assignment).
- Загрузки проверяются. Подробнее — навык `security`.

## cleanup (`CLN`)

- Мёртвый код, неиспользуемые импорты (`ruff F401/F841`, `vulture`).
- `TODO`/`FIXME`/`HACK` без задач.
- Закомментированный код.
- Дублирование логики.

## legacy (`LEG`)

- Deprecated зависимости/импорты.
- Незавершённые рефакторинги, старые и новые паттерны рядом.
- Устаревшие настройки/API.

## deploy (`DEP`)

- Settings разделены по окружениям; секреты из env.
- Security headers включены.
- Dockerfile (multi-stage, non-root, `.dockerignore`).
- CI/CD; миграции (`migrate`) в деплое.
- Health-check, graceful shutdown.
- Логи/метрики.

## tests (`TST`)

- Тесты с реальными assertion (не `assert True`).
- BDD — основное покрытие; unit — пробелы (см. `pytest-bdd`, `python-testing`).
- Моки с проверкой поведения; изоляция; без реальной БД в unit.
- Покрыт критичный код.

## tasks (`TSK`) — Django Tasks

- Постановка задач — только в `transaction.on_commit`.
- Тело задачи тонкое (вызывает сервис), тестируется unit.
- Критичные задачи идемпотентны.
- Ошибки/повторы — настройки backend'а, не бизнес-код.
- Нет Celery (в проекте используется Django Tasks).

## Формат находки

```markdown
### [SEC-001] SQL-инъекция в поиске

**Файл:** `dashboard/views.py:42`
**Тип:** A03:2021 Injection
**Проблема:** f-строка в SQL.
**Почему важно:** чтение/изменение данных БД.
**Решение:** параметризовать запрос / использовать ORM.
```

## Принципы

- Читать код, не угадывать; проверять паттерн везде.
- Конкретика: путь, строка, «было/стало».
- Критичное первым; не править сам — находить и рекомендовать.
- Без эмодзи.

## Чек-лист

- [ ] Линза(ы) выбраны; reference прочитан.
- [ ] Код прочитан (Grep + Read), не по памяти.
- [ ] Находки с путём/строкой и решением.
- [ ] Приоритизация CRITICAL/HIGH/MEDIUM/LOW.
- [ ] Отмечены положительные моменты.
