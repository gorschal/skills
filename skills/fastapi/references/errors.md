# Обработка ошибок FastAPI

Философия: **let it crash** — сервис выбрасывает доменные исключения, глобальные
handlers превращают их в ответы. Роутер не ловит бизнес-ошибки.

## Доменные исключения

```python
class DomainError(Exception):
    """Базовая ошибка домена."""

class PaymentNotFoundError(DomainError):
    def __init__(self, payment_id: UUID) -> None:
        self.payment_id = payment_id
        super().__init__(f"Payment {payment_id} not found")
```

- В сервисе — только доменные исключения. `HTTPException` запрещён.
- Транспортный маппинг (статус-код) — на границе, не в бизнес-коде.

## Глобальные handlers

```python
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse
from fastapi.exceptions import RequestValidationError
from starlette.exceptions import HTTPException as StarletteHTTPException

app = FastAPI()

@app.exception_handler(DomainError)
async def domain_error_handler(request: Request, exc: DomainError) -> JSONResponse:
    status = {PaymentNotFoundError: 404}.get(type(exc), 400)
    return JSONResponse(
        status_code=status,
        content={"code": type(exc).__name__, "detail": str(exc)},
    )

@app.exception_handler(RequestValidationError)
async def validation_error_handler(request: Request, exc: RequestValidationError) -> JSONResponse:
    # не включать exc.body — может содержать чувствительные данные
    return JSONResponse(status_code=422, content={"code": "validation_error", "detail": "Invalid request"})

@app.exception_handler(StarletteHTTPException)
async def http_error_handler(request: Request, exc: StarletteHTTPException) -> JSONResponse:
    return JSONResponse(status_code=exc.status_code, content={"code": "http_error", "detail": str(exc.detail)})

@app.exception_handler(Exception)
async def unhandled_error_handler(request: Request, exc: Exception) -> JSONResponse:
    logger.exception("unhandled_error")
    return JSONResponse(status_code=500, content={"code": "internal_error", "detail": "Internal Server Error"})
```

- Handler для `Exception` обязателен — иначе утечёт стектрейс.
- Регистрировать handler на `StarletteHTTPException`, чтобы поймать и FastAPI-исключения.
- Клиенту — безопасное сообщение; детали — в логах.

## Схема ответа об ошибке

```python
from pydantic import BaseModel

class ErrorResponse(BaseModel):
    code: str
    detail: str
    request_id: str | None = None
```

Документировать коды ошибок в OpenAPI:

```python
@router.get("/{payment_id}", response_model=PaymentOut, responses={
    404: {"model": ErrorResponse, "description": "Payment not found"},
    500: {"model": ErrorResponse, "description": "Internal server error"},
})
```

## request_id и middleware

```python
import uuid
import structlog
from starlette.middleware.base import BaseHTTPMiddleware

class RequestContextMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request, call_next):
        request_id = request.headers.get("X-Request-ID", str(uuid.uuid4()))
        structlog.contextvars.bind_contextvars(request_id=request_id)
        request.state.request_id = request_id
        logger.info("http_request_started", method=request.method, path=request.url.path)
        try:
            response = await call_next(request)
            response.headers["X-Request-ID"] = request_id
            logger.info("http_request_finished", status_code=response.status_code)
            return response
        finally:
            structlog.contextvars.clear_contextvars()
```

- Middleware — контекст/логи/тайминги; handlers — форматирование ошибок.
- `clear_contextvars()` в `finally` обязательно.
- `request_id` можно включить в тело ошибки.

## Обработка в слоях

| Слой | Действие |
|---|---|
| Service | Выбрасывает доменные исключения |
| Router | Пробрасывает (без `try/except` бизнес-ошибок) |
| Exception handler | Маппит исключение в HTTP-ответ |
| Middleware | Добавляет контекст (`request_id`, timing) |

## Антипаттерны

| ❌ | ✅ |
|---|---|
| `HTTPException` в сервисе | Доменное исключение |
| `try/except` бизнес-ошибок в роутере | Пробросить выше |
| Нет handler для `Exception` | Обязательный общий handler |
| Стектрейс/`exc.body` в ответе | Безопасное сообщение + лог |
| Ручной `JsonResponse` с ошибкой в сервисе | Централизованный handler |
| Свой формат ошибки в каждом роуте | Единая схема `ErrorResponse` |

## Чек-лист

- [ ] Сервис выбрасывает доменные исключения, не `HTTPException`.
- [ ] Зарегистрированы handlers: `DomainError`, `RequestValidationError`,
  `StarletteHTTPException`, `Exception`.
- [ ] Стектрейс и `exc.body` не уходят клиенту.
- [ ] Единая схема ответа об ошибке.
- [ ] `request_id` добавляется и включается в ошибку.
- [ ] `structlog.contextvars` очищается в `finally`.
