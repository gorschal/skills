# Ошибки и логирование

Философия: **let it crash** — не быть defensive, исключения пробрасываются
естественно и обрабатываются на границе. Плюс структурированное логирование
(structlog).

## Доменные исключения

```python
class DomainError(Exception):
    """Базовая ошибка домена."""

class UserNotFoundError(DomainError):
    def __init__(self, user_id: int) -> None:
        self.user_id = user_id
        super().__init__(f"User {user_id} not found")

class UserAlreadyExistsError(DomainError):
    def __init__(self, email: str) -> None:
        super().__init__(f"User {email} already exists")
```

- Наследники `DomainError` — семантичные, не «generic».
- Могут нести полезные атрибуты (`user_id`, `code`), но не HTTP-статусы.
- В сервисе — только доменные исключения. `HTTPException`/`JsonResponse` запрещены.

## raise low, catch high

```python
# ❌ defensive: скрывает ошибки
async def get_user(user_id: int):
    try:
        user = await service.get(user_id)
        if not user:
            raise HTTPException(404)
        return user
    except DatabaseError:
        raise HTTPException(500)

# ✅ исключение — на месте, обработка — на границе
async def get_user(user_id: int):
    user = await service.get(user_id)
    if user is None:
        raise UserNotFoundError(user_id)
    return user
```

## Ловить — только когда нужно

| Ситуация          | Пример                                    |
| ----------------- | ----------------------------------------- |
| Повторить         | `tenacity.retry` для транзиентных сбоев   |
| Трансформировать  | обернуть ошибку стороннего SDK в доменную |
| Очистить ресурсы  | `finally` / context manager               |
| Добавить контекст | `raise DomainError(...) from original`    |

Во всех остальных случаях — пробрасывать.

## Сохранение причины

```python
try:
    return self.user_repo.get(pk=user_id)
except User.DoesNotExist as e:
    raise UserNotFoundError(user_id) from e
```

`from e` сохраняет traceback и цепочку причин.

## Централизованный маппинг

Доменное исключение → транспортный ответ в одном месте:

- **FastAPI**: `@app.exception_handler(DomainError)` + handler для
  `RequestValidationError` и `Exception` (без утечки стектрейса); формат —
  RFC 7807 (`application/problem+json`, навык `api-design`).
- **Django**: middleware, ловящее `DomainError` → JSON-ответ (тот же формат).
- **FastStream/NATS**: `AckPolicy` → ack/nack/reject.

```python
# FastAPI, схематично; problem(...) формирует RFC 7807
@app.exception_handler(DomainError)
async def domain_error_handler(request: Request, exc: DomainError) -> JSONResponse:
    status = {UserNotFoundError: 404}.get(type(exc), 400)
    return problem(request, status=status, code=type(exc).__name__.upper(), detail=str(exc))

@app.exception_handler(Exception)
async def unhandled_handler(request: Request, exc: Exception) -> JSONResponse:
    logger.exception("unhandled_error")
    return problem(request, status=500, code="INTERNAL_SERVER_ERROR", detail="Internal Server Error")
```

- Клиенту — безопасное сообщение; детали — только в логах.
- Каждому доменному исключению — явный статус-код.

## Обёртка сторонних SDK

```python
class ExternalServiceError(DomainError):
    def __init__(self, service: str, original: Exception) -> None:
        super().__init__(f"{service} unavailable")
        self.__cause__ = original

try:
    ...
except httpx.HTTPError as e:
    raise ExternalServiceError("payment-api", e) from e
```

## Логирование (structlog)

```python
import logging
import structlog
from config import settings

def setup_logging() -> None:
    structlog.configure(
        processors=[
            structlog.contextvars.merge_contextvars,
            structlog.processors.add_log_level,
            structlog.processors.TimeStamper(fmt="iso", utc=True),
            structlog.processors.JSONRenderer() if not settings.debug
            else structlog.dev.ConsoleRenderer(colors=True),
        ],
        wrapper_class=structlog.make_filtering_bound_logger(logging.INFO),
    )

logger = structlog.get_logger(__name__)
```

### Правила событий

| ✅ Правильно                                    | ❌ Запрещено                  |
| ----------------------------------------------- | ----------------------------- |
| `logger.info("user_created", user_id=id)`       | `print(f"User {id} created")` |
| `logger.warning("cache_miss", key=k)`           | `logger.info("User Created")` |
| `logger.exception("payment_failed")` в `except` | `logger.info(f"...")`         |
| `snake_case`, прошедшее время                   | пробелы, заглавные            |

- Параметры — отдельными `key=value`, не в строку.
- `logger.exception` — только внутри `except` (добавляет traceback).
- Не логировать секреты, токены, пароли, PII без маскирования.

### Контекст запроса

```python
class LoggingMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        request_id = request.headers.get("X-Request-ID", str(uuid.uuid4()))
        structlog.contextvars.bind_contextvars(request_id=request_id)
        logger.info("http_request_started", method=request.method, path=request.url.path)
        try:
            response = await call_next(request)
            logger.info("http_request_finished", status_code=response.status_code)
            return response
        finally:
            structlog.contextvars.clear_contextvars()
```

`clear_contextvars()` в `finally` — обязательно, иначе контекст утечёт между
запросами/задачами.

## Антипаттерны

| ❌                            | ✅                                |
| ----------------------------- | --------------------------------- |
| `except Exception: pass`      | конкретное исключение + обработка |
| `except:`                     | `except SpecificError as e:`      |
| `raise NewError()` без `from` | `raise NewError() from e`         |
| `HTTPException` в сервисе     | доменное исключение               |
| Стектрейс в ответе клиенту    | общий handler + лог               |
| f-строки/`print`              | `logger.info("event", key=value)` |
| `logger.info(f"...")`         | параметры отдельно                |

## Чек-лист

- [ ] Доменные исключения семантичны и наследуют `DomainError`.
- [ ] В сервисе нет `HTTPException`.
- [ ] Маппинг централизован; есть handler для `Exception`.
- [ ] `raise ... from` сохраняет причину.
- [ ] Логи структурированы, события `snake_case`.
- [ ] `contextvars` очищаются в `finally`.
- [ ] Секреты не попадают в логи и ответы.
