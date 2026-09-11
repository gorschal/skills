# Обработка ошибок FastAPI

Философия: **let it crash** — сервис выбрасывает доменные исключения, глобальные
handlers превращают их в ответы. Роутер не ловит бизнес-ошибки.

Единый формат ошибки — **RFC 7807** (`application/problem+json`); канонический
контракт — в навыке `api-design`, `references/errors.md`.

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

## Схема ошибки (RFC 7807)

```python
from pydantic import BaseModel

class FieldError(BaseModel):
    field: str
    code: str
    message: str

class ProblemDetail(BaseModel):
    type: str = "about:blank"
    title: str
    status: int
    detail: str | None = None
    instance: str | None = None
    code: str
    request_id: str | None = None
    errors: list[FieldError] | None = None
```

Ответ: `Content-Type: application/problem+json`.

```json
{
  "type": "https://api.example.com/errors/resource-not-found",
  "title": "Resource Not Found",
  "status": 404,
  "detail": "Payment 123 does not exist",
  "instance": "/v1/payments/123",
  "code": "RESOURCE_NOT_FOUND",
  "request_id": "req_abc123"
}
```

## Глобальные handlers

```python
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse
from fastapi.encoders import jsonable_encoder
from fastapi.exceptions import RequestValidationError
from starlette.exceptions import HTTPException as StarletteHTTPException

app = FastAPI()

def problem(request: Request, *, status: int, code: str, title: str,
            detail: str | None = None, errors: list[FieldError] | None = None) -> JSONResponse:
    body = ProblemDetail(
        type=f"https://api.example.com/errors/{code.lower()}",
        title=title, status=status, detail=detail,
        instance=request.url.path, code=code,
        request_id=getattr(request.state, "request_id", None), errors=errors,
    )
    return JSONResponse(status, jsonable_encoder(body, exclude_none=True),
                        media_type="application/problem+json")

@app.exception_handler(DomainError)
async def domain_error_handler(request: Request, exc: DomainError) -> JSONResponse:
    status = {PaymentNotFoundError: 404}.get(type(exc), 400)
    return problem(request, status=status, code=type(exc).__name__.upper(), title="Domain Error", detail=str(exc))

@app.exception_handler(RequestValidationError)
async def validation_error_handler(request: Request, exc: RequestValidationError) -> JSONResponse:
    errors = [
        FieldError(field=".".join(str(p) for p in e["loc"][1:]), code=e["type"].upper(), message=e["msg"])
        for e in exc.errors()
    ]
    # не включать exc.body — может содержать чувствительные данные
    return problem(request, status=422, code="VALIDATION_ERROR",
                   title="Request Validation Failed", detail="Invalid request", errors=errors)

@app.exception_handler(StarletteHTTPException)
async def http_error_handler(request: Request, exc: StarletteHTTPException) -> JSONResponse:
    return problem(request, status=exc.status_code, code="HTTP_ERROR", title="HTTP Error", detail=str(exc.detail))

@app.exception_handler(Exception)
async def unhandled_error_handler(request: Request, exc: Exception) -> JSONResponse:
    logger.exception("unhandled_error")
    return problem(request, status=500, code="INTERNAL_SERVER_ERROR",
                   title="Internal Server Error", detail="Internal Server Error")
```

- Handler для `Exception` обязателен — иначе утечёт стектрейс.
- Регистрировать handler на `StarletteHTTPException`, чтобы поймать и FastAPI-исключения.
- Клиенту — безопасное сообщение; детали — в логах.

## OpenAPI

```python
@router.get("/{payment_id}", response_model=PaymentOut, responses={
    404: {"model": ProblemDetail, "description": "Payment not found"},
    500: {"model": ProblemDetail, "description": "Internal server error"},
})
```

Документировать все коды ошибок; схема — `ProblemDetail`.

## request_id и middleware

```python
import uuid
import structlog
from starlette.middleware.base import BaseHTTPMiddleware

class RequestContextMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request, call_next):
        request_id = request.headers.get("X-Request-ID", f"req_{uuid.uuid4().hex[:12]}")
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
- `request_id` — в заголовке и теле ошибки.

## Обработка в слоях

| Слой              | Действие                                      |
| ----------------- | --------------------------------------------- |
| Service           | Выбрасывает доменные исключения               |
| Router            | Пробрасывает (без `try/except` бизнес-ошибок) |
| Exception handler | Маппит исключение в problem+json              |
| Middleware        | Добавляет контекст (`request_id`, timing)     |

## Антипаттерны

| ❌                                   | ✅                               |
| ------------------------------------ | -------------------------------- |
| `HTTPException` в сервисе            | Доменное исключение              |
| `try/except` бизнес-ошибок в роутере | Пробросить выше                  |
| Нет handler для `Exception`          | Обязательный общий handler       |
| `200` с телом ошибки                 | Корректный статус + problem+json |
| Стектрейс/`exc.body` в ответе        | Безопасное сообщение + лог       |
| Свой формат ошибки в каждом роуте    | Единый `ProblemDetail`           |
| `application/json` для ошибок        | `application/problem+json`       |

## Чек-лист

- [ ] Сервис выбрасывает доменные исключения, не `HTTPException`.
- [ ] Зарегистрированы handlers: `DomainError`, `RequestValidationError`,
      `StarletteHTTPException`, `Exception`.
- [ ] Единый формат RFC 7807 (`application/problem+json`) с `code`/`request_id`.
- [ ] Стектрейс и `exc.body` не уходят клиенту.
- [ ] Все коды ошибок задокументированы в OpenAPI.
- [ ] `request_id` добавляется и включается в ошибку.
- [ ] `structlog.contextvars` очищается в `finally`.
