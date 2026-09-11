# Транзакции, ошибки и логирование

## Транзакции

Вся мутирующая логика — внутри `transaction.atomic()`. Транзакция — граница
бизнес-операции, а не отдельного запроса.

```python
from django.db import transaction

def checkout(self, order_id: int) -> None:
    with transaction.atomic():
        order = self.order_repo.get(pk=order_id)
        self.payment_repo.charge(order)
        self.order_repo.mark_paid(order)
    # побочные эффекты — ПОСЛЕ коммита
    self.notifier.send_receipt(order.id)
```

### Побочные эффекты после коммита

```python
from django.tasks import task

@task
def send_welcome_email(user_id: int) -> None:
    ...

with transaction.atomic():
    order = self.order_repo.create(...)
    transaction.on_commit(lambda: send_welcome_email.enqueue(order.id))
```

Правило: письма, Django Tasks, публикация событий — только в `on_commit`.
Иначе при откате транзакции уйдёт «письмо об откате».

### Блокировки

```python
with transaction.atomic():
    account = Account.objects.select_for_update().get(pk=account_id)
    account.balance -= amount
    account.save()
```

`select_for_update()` — для конкурентных мутаций; использовать внутри `atomic()`.

### Антипаттерны

| ❌                                      | ✅                           |
| --------------------------------------- | ---------------------------- |
| `send_email.enqueue()` внутри `atomic()` | `transaction.on_commit(...)` |
| `commit()`/`rollback()` вручную         | `atomic()` как контекст      |
| Долгие внешние вызовы внутри транзакции | вызов вне транзакции         |
| Транзакция во view                      | транзакция в сервисе         |

## Django Tasks

Фоновые задачи — через **Django Tasks** (`django.tasks`), Celery не используется.

```python
from django.tasks import task

@task
def send_welcome_email(user_id: int) -> None:
    ...

send_welcome_email.enqueue(user_id)
```

- Постановка задачи — только внутри `transaction.on_commit(...)`.
- Тело задачи тонкое: вызывает сервис; тестируется unit-тестом.
- Критичные задачи идемпотентны (повторный запуск не создаёт дубль).
- Повторы/очереди/приоритеты — настройки backend'а задач, не бизнес-код.

## Доменные исключения

```python
# exceptions.py
class DomainError(Exception):
    """Базовая ошибка домена."""

class UserAlreadyExistsError(DomainError):
    def __init__(self, email: str) -> None:
        super().__init__(f"User {email} already exists")

class UserNotFoundError(DomainError):
    def __init__(self, user_id: int) -> None:
        super().__init__(f"User {user_id} not found")
```

Сервис выбрасывает только доменные исключения; транспортный маппинг — на границе.

## Маппинг ORM-исключений

```python
from django.core.exceptions import ObjectDoesNotExist

def get_by_id(self, user_id: int) -> User:
    try:
        return self.user_repo.get(pk=user_id)
    except User.DoesNotExist as e:
        raise UserNotFoundError(user_id) from e
```

`DoesNotExist` не протекает наружу как «сырое» исключение; конвертируется в
доменное с сохранением причины (`from e`).

## Middleware

```python
import structlog
from django.http import JsonResponse

logger = structlog.get_logger(__name__)

class DomainErrorMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response

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

- Клиенту — безопасное сообщение; детали и стектрейс — только в логах.
- Доменное исключение → 400; `PermissionError` → 403; прочее → 500.

## Обработка ошибок во view

| Ситуация               | Действие                                      |
| ---------------------- | --------------------------------------------- |
| `DomainError`          | Пробросить выше — обработает middleware       |
| Ошибка валидации формы | Вернуть форму с `form.errors`                 |
| `DoesNotExist` от ORM  | Поймать в сервисе → доменное исключение (404) |
| `PermissionError`      | Пробросить выше — middleware → 403            |
| Непредвиденное         | Пробросить → middleware логирует и отдаёт 500 |

## Логирование (structlog)

```python
import logging
import structlog

def setup_logging() -> None:
    structlog.configure(
        processors=[
            structlog.contextvars.merge_contextvars,
            structlog.processors.add_log_level,
            structlog.processors.TimeStamper(fmt="iso", utc=True),
            structlog.processors.JSONRenderer() if not settings.DEBUG
            else structlog.dev.ConsoleRenderer(colors=True),
        ],
        wrapper_class=structlog.make_filtering_bound_logger(logging.INFO),
    )

logger = structlog.get_logger(__name__)
```

### Именование событий

| ✅                                              | ❌                                  |
| ----------------------------------------------- | ----------------------------------- |
| `logger.info("user_created", user_id=user.id)`  | `logger.info("User Created")`       |
| `logger.warning("cache_miss", key=cache_key)`   | `logger.info(f"User {id} created")` |
| `logger.exception("payment_failed")` в `except` | `logger.info("error")`              |

### Контекст запроса

```python
class RequestLoggingMiddleware:
    def __call__(self, request):
        structlog.contextvars.bind_contextvars(
            request_id=str(uuid.uuid4()),
            user_id=request.user.id if request.user.is_authenticated else None,
        )
        logger.info("http_request_started", path=request.path, method=request.method)
        try:
            response = self.get_response(request)
            logger.info("http_request_finished", status_code=response.status_code)
            return response
        except Exception:
            logger.exception("http_request_failed")
            raise
        finally:
            structlog.contextvars.clear_contextvars()
```

`clear_contextvars()` в `finally` — обязательно.

## Чек-лист

- [ ] Мутации в `transaction.atomic()`; побочные эффекты в `on_commit`.
- [ ] Конкурентные мутации — `select_for_update()`.
- [ ] Сервис выбрасывает доменные исключения.
- [ ] `DoesNotExist` маппится в сервисе через `from e`.
- [ ] Middleware обрабатывает `DomainError`/`PermissionError`/`Exception`.
- [ ] Стектрейс не уходит клиенту.
- [ ] Логи structlog, события `snake_case`, `contextvars` очищаются.
