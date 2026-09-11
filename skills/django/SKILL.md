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
  version: "1.0.0"
  domain: backend
  triggers: Django, ORM, QuerySet, select_related, prefetch_related, selectors, forms, migrations, manage.py, transactions, Django Tasks
  role: specialist
  scope: implementation
  output-format: code
  related-skills: python, python-testing, pytest-bdd, security, migration-safety, postgres
---

# Django

Django 5.x для server-rendered приложений: тонкие views, сервисный слой,
selectors, формы, контроль транзакций и производительности ORM.

> Общие практики языка (типизация, async, ошибки, логи, тесты, доки, инструменты)
> — в навыке `python`. Здесь только Django-специфика.

## Когда применять

- Модели, ORM-запросы, оптимизация N+1, selectors.
- Сервисный слой, транзакции, доменные исключения.
- Forms и валидация, FBV/CBV views.
- Миграции, настройки безопасности, unit-тесты Django-логики.

## Ключевые принципы

1. **Бизнес-логика — в сервисах**, не во views и не в моделях.
2. **Views тонкие**: валидация формой → вызов сервиса → ответ.
3. **Репозитории внедряются явно**, без fallback; сервис не знает про `request`.
4. **Транзакции** — `transaction.atomic()`; побочные эффекты — `on_commit`.
5. **Никаких N+1**: `select_related`/`prefetch_related` для связанных объектов.
6. **Время — UTC**: `timezone.now()`, `USE_TZ=True`.
7. **Ошибки — доменными исключениями** + middleware для маппинга на ответ.
8. **Тесты**: BDD — основное покрытие; unit — логика вне BDD и критичные ветки.

## Архитектура (5 слоёв)

| Слой | Файл | Ответственность |
|---|---|---|
| Views | `views.py` | HTTP, валидация формой, вызов сервиса, ответ |
| Services | `services.py` | Бизнес-логика, транзакции, оркестрация. Без `request`/`response` |
| Selectors | `selectors.py` | Чтение: сложные (JOIN, агрегации) или переиспользуемые запросы. Без мутаций |
| Models | `models.py` | Структура данных, простые свойства, `__str__`. Без бизнес-логики |
| Forms | `forms.py` | Только валидация данных |

### Сервис

```python
from typing import Protocol
from django.db import transaction

class UserRepository(Protocol):
    def filter(self, **kwargs) -> "QuerySet": ...
    def create_user(self, email: str, password: str) -> "User": ...

class UserService:
    def __init__(
        self,
        user_repo: UserRepository,                    # явно, без fallback
        email_service: "EmailService | None" = None,  # вторичная — fallback ок
    ) -> None:
        self.user_repo = user_repo
        self.email_service = email_service or EmailService()

    def create_user(self, email: str, password: str) -> "User":
        with transaction.atomic():
            if self.user_repo.filter(email=email).exists():
                raise UserAlreadyExistsError(email)
            user = self.user_repo.create_user(email, password)
            transaction.on_commit(lambda: self.email_service.send_welcome(user.id))
            return user
```

- Вся мутирующая логика — внутри `transaction.atomic()`.
- Письма/Django Tasks — в `transaction.on_commit()`, не до коммита.
- Сервис не обращается к `request`; принимает данные (`user_id`), не объекты запроса.
- **Всё сохранение — через репозиторий.** Прямой `user.save()` в сервисе ломает
  изоляцию unit-тестов (нужна БД) — запрещено.

### Selectors

```python
def get_orders_with_items(user_id: int) -> "QuerySet[Order]":
    """Заказы пользователя со связанными данными (без N+1)."""
    return (
        Order.objects.filter(user_id=user_id)
        .select_related("customer")
        .prefetch_related(
            Prefetch("items", queryset=OrderItem.objects.select_related("product"))
        )
    )
```

- Selector — если запрос сложный (JOIN/агрегация) **или** переиспользуется ≥2 раз.
- Простые `filter(...)`/`get(pk=...)` остаются в сервисе или менеджере модели.

### Views: FBV и CBV

```python
# FBV — простой сценарий
def order_list(request: HttpRequest) -> HttpResponse:
    """Список заказов текущего пользователя с пагинацией."""
    page_obj = Paginator(get_orders_with_items(request.user.id), 20).get_page(request.GET.get("page"))
    return render(request, "orders/list.html", {"page_obj": page_obj})

# CBV — CRUD-экран
class OrderCreateView(LoginRequiredMixin, CreateView):
    model = Order
    form_class = OrderForm
    success_url = reverse_lazy("orders:list")

    def form_valid(self, form: OrderForm) -> HttpResponse:
        self.object = OrderService(order_repo=Order.objects).create(
            user_id=self.request.user.id, **form.cleaned_data
        )
        return redirect(self.get_success_url())
```

- FBV — простые сценарии; CBV — CRUD-экраны. Логика в обоих случаях в сервисе.
- `get_queryset()` вместо статического `queryset`, когда нужна динамика/права.
- Проверять права (`LoginRequiredMixin`/`@login_required`), не отдавать чужие объекты.

Подробно: [references/architecture.md](references/architecture.md).

## ORM и производительность

| ✅ Правильно | ❌ Запрещено |
|---|---|
| `Order.objects.select_related("customer")` | `Order.objects.all()` + доступ в цикле |
| `.prefetch_related("items__product")` | `order.customer.name` в цикле (N+1) |
| `only()`/`defer()` для части полей | тянуть все поля без нужды |
| `annotate()`/`aggregate()`, `F()`, `Q()` | агрегация в Python |
| `bulk_create`/`bulk_update` | `save()` в цикле |
| пагинация (`Paginator`) | `.all()` без пагинации в ответе |

```python
# вложенные связи
orders = Order.objects.prefetch_related("items__product", "customer__profile")

# массовые операции
Product.objects.bulk_create(items, batch_size=1000)
Product.objects.filter(category=old).update(category=new)

# F() — операция на стороне БД
Product.objects.update(price=F("price") * 1.1)
```

Подробно: [references/orm.md](references/orm.md).

## Работа со временем

```python
# base.py
USE_TZ = True
TIME_ZONE = "UTC"

from django.utils import timezone
now = timezone.now()      # ✅
datetime.now()            # ❌ без tz
```

## Обработка ошибок

```python
# exceptions.py
class DomainError(Exception): ...

class UserAlreadyExistsError(DomainError): ...

# middleware.py
class DomainErrorMiddleware:
    def __call__(self, request):
        try:
            return self.get_response(request)
        except DomainError as e:
            return JsonResponse({"error": str(e), "code": type(e).__name__}, status=400)
        except PermissionError:
            return JsonResponse({"error": "Forbidden"}, status=403)
        except Exception:
            logger.exception("unhandled_error")
            return JsonResponse({"error": "Internal Server Error"}, status=500)
```

- `DomainError` пробрасывается из сервиса, обрабатывается middleware.
- `DoesNotExist` маппится в сервисе в доменное исключение (`raise ... from e`).
- Ошибки валидации формы — возвращать форму с `form.errors`.
- Стектрейс клиенту не отдавать.

Подробно: [references/transactions-errors.md](references/transactions-errors.md).

## Формы и валидация

```python
class SignupForm(forms.ModelForm):
    class Meta:
        model = User
        fields = ["email", "username"]

    def clean_email(self) -> str:
        email = self.cleaned_data["email"].lower()
        if User.objects.filter(email__iexact=email).exists():
            raise forms.ValidationError("Email уже занят")
        return email
```

- Вся валидация — в форме (`clean_<field>`, `clean`), не во view.
- `ModelForm` где возможно; `fields` перечислять явно, не `"__all__"`.
- View только проверяет `form.is_valid()` и вызывает сервис.

## Миграции

```bash
python manage.py makemigrations
python manage.py sqlmigrate app 0004        # посмотреть реальный SQL
python manage.py makemigrations --check --dry-run   # CI
```

- Данные — только в `RunPython`, всегда с `reverse_code`.
- В `RunPython` — `apps.get_model(...)`, не прямой импорт модели.
- Схему и данные разделять на разные миграции.
- Применённые на проде миграции не редактировать.
- `atomic = False`, concurrent-индексы, батч-backfill, zero-downtime —
  в навыке `migration-safety`.

Подробно: [references/migrations.md](references/migrations.md).

## Логирование

```python
import structlog
logger = structlog.get_logger(__name__)

logger.info("user_created", user_id=user.id)
logger.exception("payment_failed", payment_id=str(payment_id))
```

- structlog; события `snake_case` прошедшего времени; параметры отдельными
  `key=value`; без `print` и f-строк.
- `request_id`/`user_id` — через `structlog.contextvars`, очищать в `finally`.

## Тестирование

Политика: **BDD — основное сквозное покрытие**; unit — логика вне BDD и
критичные ветки. Unit не дублирует BDD.

- pytest-функции и фикстуры, **не** `unittest.TestCase`/`django.test.TestCase`.
- Без БД, HTTP и `TestClient`: репозиторий — `Mock`, транзакции — патчатся.

```python
from unittest.mock import Mock, patch
import pytest

@patch("app.services.transaction.atomic")
@patch("app.services.transaction.on_commit")
def test_create_user_raises_when_email_taken(mock_on_commit, mock_atomic) -> None:
    repo = Mock()
    repo.filter.return_value.exists.return_value = True
    service = UserService(user_repo=repo)

    with pytest.raises(UserAlreadyExistsError):
        service.create_user("taken@example.com", "password123")
```

Что тестируем: бизнес-логику сервисов, тела сигналов и задач, сложные
валидаторы, логику вне BDD. Что нет: поля моделей, `__str__`, стандартную
валидацию форм, HTTP/views (BDD).

Подробно: [references/testing.md](references/testing.md).

## Документирование

- Django views — **максимально подробно**: поведение, вход/выход, побочные
  эффекты, делегирование в сервис (аудитория — мейнтейнер, язык RU).
- Сервисы — Google-style: «почему» + ограничения, `Args/Returns/Raises/Side Effects`.
- Selectors — смысл запроса и особенности.
- Тесты — кратко, одной строкой для нетривиального кейса.
- Не документировать `__init__`, `__str__`, одно-строчные геттеры, простой CRUD.

Общая матрица — в навыке `python`, `references/documentation.md`.

## Безопасность

- `DEBUG=False`, `ALLOWED_HOSTS` — только нужные домены, `SECRET_KEY` из env.
- HTTPS: `SECURE_SSL_REDIRECT`, `SESSION_COOKIE_SECURE`, `CSRF_COOKIE_SECURE`,
  `SECURE_HSTS_SECONDS > 0`, `X_FRAME_OPTIONS="DENY"`.
- CSRF: не использовать `@csrf_exempt` без причины; `{% csrf_token %}` в формах.
- XSS: не использовать `|safe`/`mark_safe` на пользовательских данных.
- SQL: только ORM/параметры; `raw`/`extra` с f-строками запрещены.
- IDOR: `get_queryset()` фильтрует по `request.user`; проверка владельца.
- Mass assignment: `fields` перечислять, не `"__all__"`.
- Загрузки: проверять тип, размер, имя; хранить вне кода.
- Не логировать пароли/токены; `AUTH_PASSWORD_VALIDATORS` не пустой.

Подробно: [references/security.md](references/security.md).

## Инструменты

```bash
docker compose exec django uv run python manage.py test
docker compose exec django uv run python manage.py makemigrations --check --dry-run
docker compose exec django uv run ruff check . && docker compose exec django uv run ruff format --check .
docker compose exec django uv run pyright
```

## Запрещённые паттерны

| ❌ Запрещено | ✅ Правильно |
|---|---|
| Бизнес-логика во view | Вызов `UserService.create_user()` |
| Бизнес-логика в модели | Сервис |
| `fallback` на репозиторий в `__init__` | Явная передача репозитория |
| `user.save()` в сервисе | `self.user_repo.create_user(...)` |
| `send_email.enqueue()` внутри транзакции | `transaction.on_commit(...)` |
| `Order.objects.all()` в цикле | `select_related`/`prefetch_related` |
| `.all()` без пагинации в ответе | `Paginator` |
| `datetime.now()` | `timezone.now()` |
| `print(f"...")` | `logger.info("event", key=value)` |
| `fields = "__all__"` | Явный список полей |
| `@csrf_exempt` без причины | CSRF-защита |
| `unittest.TestCase` в unit-тестах | pytest-функции/фикстуры |

## Чек-лист code review

- [ ] Бизнес-логика в сервисах, views тонкие, модели без логики.
- [ ] Репозитории внедрены явно (без fallback); сервис не знает про `request`.
- [ ] Вся мутация — в `transaction.atomic()`; побочные эффекты в `on_commit`.
- [ ] Нет прямого `save()` в сервисе.
- [ ] Нет N+1; сложные/переиспользуемые запросы — в selectors.
- [ ] Пагинация на списочных эндпоинтах.
- [ ] `timezone.now()`, `USE_TZ=True`.
- [ ] `DomainError` → middleware; `DoesNotExist` маппится в сервисе.
- [ ] Валидация — в формах; `fields` без `"__all__"`.
- [ ] Миграции: `sqlmigrate` проверен, `RunPython` с `reverse_code`.
- [ ] Настройки безопасности (DEBUG, ALLOWED_HOSTS, cookies, HSTS).
- [ ] Unit-тесты не дублируют BDD; репозитории/транзакции замоканы.
- [ ] Логи structlog, события `snake_case`; секреты не логируются.
- [ ] Docstring Django views подробный; типизация присутствует.
- [ ] `ruff check`, `ruff format --check`, `pyright`, тесты проходят.

## Справочники

| Тема | Reference | Загружать когда |
|---|---|---|
| Архитектура, слои, DI, forms, views | [references/architecture.md](references/architecture.md) | Проектирование сервисов/selectors/views |
| ORM и производительность | [references/orm.md](references/orm.md) | Модели, N+1, managers, пагинация |
| Транзакции, ошибки, логи | [references/transactions-errors.md](references/transactions-errors.md) | atomic/on_commit, доменные исключения, structlog |
| Миграции | [references/migrations.md](references/migrations.md) | makemigrations, RunPython, безопасные изменения |
| Безопасность | [references/security.md](references/security.md) | settings, CSRF/XSS/SQLi, IDOR, загрузки |
| Тестирование | [references/testing.md](references/testing.md) | unit-тесты сервисов, моки, границы BDD |

## Связанные навыки

- `python` — общие практики языка (типизация, ошибки, логи, доки, инструменты).
- `python-testing` / `pytest-bdd` — тестовая инфраструктура и BDD.
- `security` — расширенный чек-лист безопасности.
- `migration-safety` — zero-downtime миграции, concurrent-индексы, backfill.
- `postgres` — индексы, планы, производительность БД.
