---
name: django
description: >
  Use when working with Django 5.x server-rendered applications: models and ORM
  (N+1, select_related/prefetch_related), selectors, services, forms, FBV/CBV
  views, transactions, migrations, security, unit tests. Триггеры: Django,
  manage.py, models.py, QuerySet, select_related, prefetch_related, selectors,
  forms, migrations, makemigrations, atomic, on_commit, Django Tasks, enqueue,
  django-structlog.
  Общие практики Python — навык python; FastAPI — fastapi.
license: MIT
compatibility: opencode
metadata:
  version: "1.1.0"
  domain: backend
  triggers: Django, ORM, QuerySet, select_related, prefetch_related, selectors, forms, migrations, manage.py, transactions, Django Tasks
  role: specialist
  scope: implementation
  output-format: code
  related-skills: python, python-testing, pytest-bdd, security, migration-safety, postgres
---

# Django

Django 5.x для server-rendered приложений: тонкие views, сервисный слой,
selectors, формы, транзакции, производительность ORM.

> Общие практики языка — навык `python`. Здесь только Django-специфика.

## Когда применять

- Модели, ORM, N+1, selectors; сервисный слой, транзакции.
- Forms и валидация, FBV/CBV; миграции, безопасность, unit-тесты.

## Ключевые принципы

1. **Бизнес-логика — в сервисах**, не во views/моделях.
2. **Views тонкие**: валидация формой → вызов сервиса → ответ.
3. **Репозитории внедряются явно**, без fallback; сервис не знает про `request`.
4. **Транзакции** — `transaction.atomic()`; побочные эффекты — `on_commit`.
5. **Никаких N+1**: `select_related`/`prefetch_related`.
6. **Время — UTC**: `timezone.now()`, `USE_TZ=True`.
7. **Ошибки — доменными исключениями** + middleware.
8. **Тесты**: BDD — основное покрытие; unit — пробелы и критичные ветки.

## Архитектура (5 слоёв)

| Слой | Файл | Ответственность |
|---|---|---|
| Views | `views.py` | HTTP, валидация формой, вызов сервиса, ответ |
| Services | `services.py` | бизнес-логика, транзакции, оркестрация. Без `request` |
| Selectors | `selectors.py` | чтение: сложные/переиспользуемые запросы. Без мутаций |
| Models | `models.py` | данные, простые свойства, `__str__`. Без бизнес-логики |
| Forms | `forms.py` | только валидация данных |

### Сервис

```python
def create_user(self, email: str, password: str) -> User:
    with transaction.atomic():
        if self.user_repo.filter(email=email).exists():
            raise UserAlreadyExistsError(email)
        user = self.user_repo.create_user(email, password)
        transaction.on_commit(lambda: self.email_service.send_welcome(user.id))
        return user
```

- Вся мутация — в `atomic()`; письма/задачи — в `on_commit`.
- Сервис не знает про `request`; принимает данные (`user_id`).
- **Всё сохранение — через репозиторий**; `user.save()` в сервисе запрещён.

### Selectors

```python
def get_orders_with_items(user_id: int) -> QuerySet[Order]:
    return (Order.objects.filter(user_id=user_id).select_related("customer")
            .prefetch_related("items__product"))
```

Selector — если запрос сложный (JOIN/агрегация) **или** переиспользуется ≥2 раз.
Простые `filter`/`get` — в сервисе/менеджере.

### Views: FBV и CBV

```python
# FBV: Paginator(get_orders_with_items(request.user.id), 20) → render(...)
# CBV: form_valid → OrderService(...).create(user_id=request.user.id, **form.cleaned_data)
```

- FBV — простые сценарии; CBV — CRUD. Логика — в сервисе.
- `get_queryset()` для динамики/прав; проверять права; не отдавать чужие объекты.

Подробно: [references/architecture.md](references/architecture.md).

## ORM и производительность

| ✅ | ❌ |
|---|---|
| `select_related("customer")` | `.all()` + доступ в цикле |
| `.prefetch_related("items__product")` | `order.customer.name` в цикле (N+1) |
| `only()`/`defer()` | тянуть все поля |
| `annotate`/`aggregate`, `F()`, `Q()` | агрегация в Python |
| `bulk_create`/`bulk_update` | `save()` в цикле |
| `Paginator` | `.all()` без пагинации |

Подробно: [references/orm.md](references/orm.md).

## Время, ошибки, формы

`USE_TZ=True`, `TIME_ZONE="UTC"`; в коде — `timezone.now()` (не `datetime.now()`).

```python
class DomainError(Exception): ...

# middleware.py — JSON-клиентам RFC 7807, браузеру HTML
class DomainErrorMiddleware:
    def __call__(self, request):
        try:
            return self.get_response(request)
        except DomainError as e:
            return self._render(request, 400, type(e).__name__.upper(), str(e))
        except Exception:
            logger.exception("unhandled_error")
            return self._render(request, 500, "INTERNAL_SERVER_ERROR", "Internal Server Error")
```

- **Content negotiation**: JSON-клиентам — RFC 7807 (`api-design`); браузеру — HTML.
- `DoesNotExist` маппится в сервисе (`raise ... from e`); ошибки формы — `form.errors`.
- Валидация — в форме (`clean_<field>`), не во view; `fields` без `"__all__"`.

Подробно: [references/transactions-errors.md](references/transactions-errors.md).

## Миграции

```bash
python manage.py makemigrations
python manage.py sqlmigrate app 0004
python manage.py makemigrations --check --dry-run   # CI
```

- Данные — только в `RunPython`, с `reverse_code` и `apps.get_model`.
- Схему и данные — в разных миграциях; применённые не редактировать.
- `atomic=False`, concurrent-индексы, backfill, zero-downtime — `migration-safety`.

Подробно: [references/migrations.md](references/migrations.md).

## Логирование

- structlog; события `snake_case` прошедшего времени, `key=value`; без `print`/f-строк.
- `request_id`/`user_id` — через `structlog.contextvars`, очищать в `finally`.

## Тестирование

Политика: **BDD — основное покрытие**; unit — пробелы и критичные ветки.

- pytest-функции/фикстуры, **не** `unittest.TestCase`/`django.test.TestCase`.
- Без БД/HTTP: репозиторий — `Mock`, `transaction.atomic`/`on_commit` — патчить.

Тестируем: логику сервисов, тела сигналов/задач, сложные валидаторы. Не тестируем:
поля моделей, `__str__`, стандартную валидацию форм, HTTP/views (BDD).

Подробно: [references/testing.md](references/testing.md).

## Документирование

- Django views — подробно: поведение, вход/выход, побочные эффекты (RU).
- Сервисы — Google-style: «почему» + ограничения.
- Selectors — смысл запроса; тесты — кратко.
- Не документировать `__init__`, `__str__`, геттеры, простой CRUD.

Общая матрица — навык `python`, `references/documentation.md`.

## Безопасность

- `DEBUG=False`, `ALLOWED_HOSTS` ограничен, `SECRET_KEY` из env.
- HTTPS: `SECURE_SSL_REDIRECT`, secure cookies, HSTS, `X_FRAME_OPTIONS="DENY"`.
- CSRF: без `@csrf_exempt` без причины; XSS: без `|safe`/`mark_safe` на вводе.
- SQL — только ORM/параметры; IDOR: `get_queryset()` по `request.user`.
- Mass assignment: `fields` явно; загрузки проверять; не логировать пароли/токены.

Подробно: [references/security.md](references/security.md) и навык `security`.

## Инструменты

```bash
docker compose exec django uv run python manage.py test
docker compose exec django uv run python manage.py makemigrations --check --dry-run
docker compose exec django uv run ruff check . && docker compose exec django uv run pyright
```

## Запрещённые паттерны

| ❌ | ✅ |
|---|---|
| Бизнес-логика во view/модели | сервис |
| `fallback` на репозиторий | явная передача |
| `user.save()` в сервисе | `self.user_repo.create_user(...)` |
| `send_email.enqueue()` внутри транзакции | `transaction.on_commit(...)` |
| `Order.objects.all()` в цикле | `select_related`/`prefetch_related` |
| `.all()` без пагинации | `Paginator` |
| `datetime.now()` | `timezone.now()` |
| `print(f"...")` | `logger.info("event", key=value)` |
| `fields = "__all__"` | явный список |
| `@csrf_exempt` без причины | CSRF-защита |
| `unittest.TestCase` в unit | pytest-функции |

## Чек-лист code review

- [ ] Бизнес-логика в сервисах, views тонкие, модели без логики.
- [ ] Репозитории внедрены явно; сервис не знает про `request`.
- [ ] Мутация в `atomic()`; эффекты в `on_commit`; нет `save()` в сервисе.
- [ ] Нет N+1; сложные запросы — в selectors; пагинация есть.
- [ ] `timezone.now()`, `USE_TZ=True`.
- [ ] `DomainError` → middleware; `DoesNotExist` маппится в сервисе.
- [ ] Валидация в формах; `fields` без `"__all__"`.
- [ ] Миграции: `sqlmigrate`, `RunPython` с `reverse_code`.
- [ ] Настройки безопасности (DEBUG, ALLOWED_HOSTS, cookies, HSTS).
- [ ] Unit не дублирует BDD; репозитории/транзакции замоканы.
- [ ] Логи structlog; docstring views; `ruff`/`pyright`/тесты проходят.

## Справочники

| Тема | Reference | Когда |
|---|---|---|
| Архитектура, слои, DI, forms, views | [references/architecture.md](references/architecture.md) | Проектирование |
| ORM и производительность | [references/orm.md](references/orm.md) | Модели, N+1, пагинация |
| Транзакции, ошибки, логи | [references/transactions-errors.md](references/transactions-errors.md) | atomic/on_commit, исключения |
| Миграции | [references/migrations.md](references/migrations.md) | makemigrations, RunPython |
| Безопасность | [references/security.md](references/security.md) | settings, CSRF/XSS/SQLi, IDOR |
| Тестирование | [references/testing.md](references/testing.md) | unit сервисов, моки, BDD |

## Связанные навыки

- `python` — общие практики; `python-testing`/`pytest-bdd` — тесты.
- `security` — расширенный чек-лист; `migration-safety` — миграции.
- `postgres` — индексы, планы, производительность.
