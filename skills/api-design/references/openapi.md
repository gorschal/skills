# OpenAPI

Спека генерируется из кода (FastAPI) — единственный источник правды.
Устаревшая спека хуже отсутствующей.

## Принципы

- Переиспользовать компоненты (`$ref`) для схем, ответов, параметров, security.
- Каждая операция: уникальный `operationId`, `summary`, `description`, `tags`,
  **все** ответы (включая ошибки).
- Реалистичные `examples` на схемах и ответах.
- Валидация/линт в CI.

## Генерация (FastAPI)

```python
app = FastAPI(
    title="Users API",
    version="1.0.0",
    description="Manages user accounts and profiles.",
    servers=[{"url": "https://api.example.com/v1", "description": "Production"}],
    openapi_url="/openapi.json",
    docs_url="/docs",
    redoc_url="/redoc",
)
```

```python
@router.get(
    "/users",
    response_model=Page[UserRead],
    summary="List users",
    description="Retrieve a paginated list of users with optional filtering.",
    operation_id="listUsers",
    tags=["Users"],
    responses={
        401: {"$ref": "#/components/responses/Unauthorized"},
        429: {"$ref": "#/components/responses/RateLimitExceeded"},
    },
)
async def list_users(...) -> Page[UserRead]: ...
```

- `response_model`/`responses` формируют спеку.
- `deprecated=True` помечает операцию устаревшей.

## Схемы и примеры (Pydantic v2)

```python
class UserRead(BaseModel):
    model_config = ConfigDict(from_attributes=True, json_schema_extra={
        "examples": [{"id": 123, "email": "john@example.com", "name": "John Doe"}]
    })

    id: int = Field(examples=[123])
    email: EmailStr = Field(examples=["john@example.com"])
    name: str = Field(min_length=1, max_length=100, examples=["John Doe"])
    created_at: datetime = Field(examples=["2024-01-15T10:30:00Z"])
```

- `Field(examples=[...])` — примеры полей; `json_schema_extra` — объект целиком.
- Раздельные `Create`/`Update`/`Read` модели; `extra="forbid"` для входа.
- Read-only поля — через отдельные схемы вывода.

## Теги

```python
tags_metadata = [
    {"name": "Users", "description": "User management operations"},
    {"name": "Orders", "description": "Order management"},
]
app = FastAPI(openapi_tags=tags_metadata)
```

Стабильные имена тегов (используются в доках и codegen).

## Security schemes

```python
from fastapi.security import HTTPBearer

bearer = HTTPBearer()

@router.get("/users/{id}", dependencies=[Depends(bearer)])
async def get_user(id: int) -> UserRead: ...
```

- HTTP bearer → `bearerAuth` в спеке.
- API key — `APIKeyHeader(name="X-API-Key")`; OAuth2 — `OAuth2PasswordBearer`.
- Применять глобально (`FastAPI(dependencies=[...])`) или по операции.

```yaml
components:
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
```

## Актуальность

- Генерация из кода — один источник правды.
- CI-проверки: спека валидна, нет недокументированных маршрутов, примеры
  соответствуют моделям.
- Версионировать спеку вместе с API (`/openapi.json` на версию).
- Генерировать SDK/клиенты из той же спеки.
- Ревью диффа спеки в PR как части контракта.

```bash
swagger-cli validate openapi.yaml
spectral lint openapi.yaml
```

## Антипаттерны

| ❌                                | ✅                   |
| --------------------------------- | -------------------- |
| Спека руками и расходится с кодом | генерация из кода    |
| Нет `operationId`/summary/tags    | заполнены            |
| Документирован только `200`       | все ответы           |
| Нет примеров                      | `examples` на схемах |
| Дублирование схем                 | `$ref`-компоненты    |
| Спека не проверяется              | линт в CI            |

## Чек-лист

- [ ] Спека генерируется из кода и актуальна.
- [ ] `info`/`servers` заполнены, версии учтены.
- [ ] У каждой операции `operationId`, summary, description, tags.
- [ ] Схемы запрос/ответ через `response_model` и `$ref`.
- [ ] Примеры на схемах и ответах.
- [ ] Все ошибки (`401`, `404`, `409`, `422`, `429`, `5xx`) задокументированы.
- [ ] Security schemes объявлены и применены.
- [ ] Линт спеки в CI.
