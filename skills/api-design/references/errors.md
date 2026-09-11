# Обработка ошибок (RFC 7807)

Единый формат ошибок во всём API — `application/problem+json`. Стандартные
HTTP-статусы; никаких ошибок внутри `200`.

## Формат

```http
HTTP/1.1 404 Not Found
Content-Type: application/problem+json

{
  "type": "https://api.example.com/errors/resource-not-found",
  "title": "Resource Not Found",
  "status": 404,
  "detail": "User with ID 123 does not exist",
  "instance": "/v1/users/123",
  "code": "RESOURCE_NOT_FOUND",
  "request_id": "req_abc123"
}
```

- `type` — стабильный URI класса ошибки (не generic-строка).
- `title` — краткое описание; `detail` — конкретное, actionable.
- `code` (расширение) — машинный код; `request_id` — для корреляции.
- `errors[]` — детали валидации по полям.

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

## Handler

```python
class APIError(Exception):
    def __init__(self, status: int, code: str, detail: str, title: str | None = None):
        self.status, self.code, self.detail = status, code, detail
        self.title = title or code.replace("_", " ").title()

@app.exception_handler(APIError)
async def api_error_handler(request: Request, exc: APIError) -> JSONResponse:
    body = ProblemDetail(
        type=f"https://api.example.com/errors/{exc.code.lower()}",
        title=exc.title, status=exc.status, detail=exc.detail,
        instance=request.url.path, code=exc.code,
        request_id=getattr(request.state, "request_id", None),
    )
    return JSONResponse(exc.status, body.model_dump(exclude_none=True),
                        media_type="application/problem+json")
```

## Статус-коды

| Условие                        | Статус                    | `code`                                          |
| ------------------------------ | ------------------------- | ----------------------------------------------- |
| Неверная схема/формат/диапазон | `422` (FastAPI) или `400` | `VALIDATION_ERROR`                              |
| Нет/невалидные credentials     | `401`                     | `MISSING_TOKEN`/`INVALID_TOKEN`/`EXPIRED_TOKEN` |
| Нет прав                       | `403`                     | `INSUFFICIENT_PERMISSIONS`                      |
| Нет ресурса                    | `404`                     | `RESOURCE_NOT_FOUND`                            |
| Дубликат/конфликт              | `409`                     | `RESOURCE_ALREADY_EXISTS`/`CONFLICT`            |
| Rate limit                     | `429` + `Retry-After`     | `RATE_LIMIT_EXCEEDED`                           |
| Непредвиденное                 | `500`                     | `INTERNAL_SERVER_ERROR`                         |
| Недоступно                     | `503` + `Retry-After`     | `SERVICE_UNAVAILABLE`                           |

## Валидация (Pydantic v2)

FastAPI по умолчанию отдаёт `422`; переопределить `RequestValidationError` под
problem+json:

```python
@app.exception_handler(RequestValidationError)
async def validation_handler(request: Request, exc: RequestValidationError) -> JSONResponse:
    errors = [
        FieldError(field=".".join(str(p) for p in e["loc"][1:]), code=e["type"].upper(), message=e["msg"])
        for e in exc.errors()
    ]
    body = ProblemDetail(
        type="https://api.example.com/errors/validation-error",
        title="Request Validation Failed", status=422,
        detail="One or more fields are invalid.",
        instance=request.url.path, code="VALIDATION_ERROR",
        request_id=getattr(request.state, "request_id", None), errors=errors,
    )
    return JSONResponse(422, jsonable_encoder(body, exclude_none=True),
                        media_type="application/problem+json")
```

## Коды ошибок

Каталог `code → статус → описание`; коды — часть контракта (смена — breaking).
Примеры: `VALIDATION_ERROR`, `AUTHENTICATION_ERROR`, `AUTHORIZATION_ERROR`,
`RESOURCE_NOT_FOUND`, `CONFLICT`, `RATE_LIMIT_EXCEEDED`, `INTERNAL_SERVER_ERROR`.

## Безопасность

- Не отдавать стектрейсы, SQL/ORM-ошибки, пути, значения окружения.
- `500` — общее сообщение; полный лог — на сервере.
- Не различать «неверный пароль» и «нет пользователя» (enumeration).
- `request_id` — в заголовке и теле.

## Request ID

```python
class RequestIDMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request, call_next):
        rid = request.headers.get("X-Request-ID", f"req_{uuid.uuid4().hex[:12]}")
        request.state.request_id = rid
        response = await call_next(request)
        response.headers["X-Request-ID"] = rid
        return response
```

## Retry

- Retryable: `408`, `429` (+`Retry-After`), `502`, `503`, `504`.
- Non-retryable: `400`, `401`, `403`, `404`, `409`, `422`.

## Антипаттерны

| ❌                                   | ✅                    |
| ------------------------------------ | --------------------- |
| `200` с телом ошибки                 | корректный статус     |
| Разный формат ошибок                 | RFC 7807              |
| Стектрейс/SQL в ответе               | общее сообщение + лог |
| Нет `request_id`                     | заголовок + тело      |
| Ошибки не задокументированы          | все ответы в OpenAPI  |
| Раскрытие существования пользователя | общее сообщение       |

## Чек-лист

- [ ] Единый формат `problem+json`; корректные статусы.
- [ ] `code` на каждой ошибке; `detail` actionable.
- [ ] Валидация (в т.ч. кросс-поля) в том же envelope.
- [ ] `X-Request-ID` принят/сгенерирован и возвращён.
- [ ] `500` не раскрывает внутренности.
- [ ] `Retry-After` на `429`/`503`.
