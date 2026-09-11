# REST-паттерны

## Ресурсы

- Существительные, множественное число, lowercase, дефисы для составных:
  `/users`, `/shipping-addresses`.
- Никаких глаголов/действий в пути/query (`/getUser`, `/user?action=delete`).
  Для операций — под-ресурс: `POST /orders/{id}/refunds`.
- Вложенность ≤ 2–3 уровней; глубже — top-level ресурс с фильтром.
- Фильтры/сортировка/поиск — query-параметры.
- Один стиль именования во всём API (для Python — `snake_case`).

```
GET    /v1/users
POST   /v1/users
GET    /v1/users/{id}
PUT    /v1/users/{id}          # полная замена
PATCH  /v1/users/{id}          # частичное обновление
DELETE /v1/users/{id}
GET    /v1/users/{id}/orders   # вложенная коллекция
```

## Методы

| Метод  | Safe | Idempotent | Назначение              |
| ------ | ---- | ---------- | ----------------------- |
| GET    | да   | да         | чтение                  |
| POST   | нет  | нет        | создание/действие       |
| PUT    | нет  | да         | полная замена           |
| PATCH  | нет  | нет        | частичное обновление    |
| DELETE | нет  | да         | удаление (повтор → 404) |

```python
@router.post("/users", status_code=status.HTTP_201_CREATED, response_model=UserRead)
async def create_user(payload: UserCreate) -> UserRead: ...

@router.delete("/users/{user_id}", status_code=status.HTTP_204_NO_CONTENT)
async def delete_user(user_id: int) -> None: ...
```

## Статус-коды

- **2xx**: `200` OK, `201` Created + `Location`, `202` Accepted, `204` No Content.
- **3xx**: `304` Not Modified.
- **4xx**: `400`/`422` валидация, `401` не аутентифицирован, `403` запрещено,
  `404` нет, `405` метод не разрешён, `409` конфликт, `429` rate limit.
- **5xx**: `500`, `502`, `503`, `504`.

## Фильтрация, сортировка, поиск

```
GET /v1/users?status=active&role=admin
GET /v1/products?price_min=100&price_max=500
GET /v1/users?sort=-created_at
GET /v1/users?q=john
GET /v1/users?fields=id,name,email
```

```python
@router.get("/users", response_model=Page[UserRead])
async def list_users(
    status_: Annotated[Literal["active","inactive"] | None, Query(alias="status")] = None,
    q: Annotated[str | None, Query(min_length=1, max_length=100)] = None,
    sort: Annotated[str, Query(pattern=r"^-?[a-z_]+(,-?[a-z_]+)*$")] = "-created_at",
    limit: Annotated[int, Query(ge=1, le=100)] = 20,
    offset: Annotated[int, Query(ge=0)] = 0,
) -> Page[UserRead]: ...
```

- Whitelist сортируемых/фильтруемых полей (не подставлять ввод в SQL).
- Фильтры — **до** пагинации; сортировка заканчивается уникальным tiebreaker.

## Bulk-операции

```python
class BulkItemResult(BaseModel):
    index: int
    status: int
    id: int | None = None
    error: ErrorDetail | None = None

@router.post("/users/bulk", response_model=list[BulkItemResult])
async def bulk_create(items: list[UserCreate]) -> list[BulkItemResult]: ...
```

- Отдельный endpoint; **поэлементные** результаты, не «всё-или-ничего».
- Ограничение размера батча (например, ≤ 100); превышение → `422`.

## Идемпотентность

- Клиент шлёт `Idempotency-Key: <uuid>` на небезопасные POST.
- Сервер хранит ключ → ответ на TTL; повтор возвращает тот же ответ без повтора.
- Ключ скоупится на пользователя/маршрут; другой payload с тем же ключом → `409`.

```python
@router.post("/payments", status_code=201)
async def create_payment(
    payload: PaymentCreate,
    idempotency_key: Annotated[str, Header(alias="Idempotency-Key", min_length=16)],
) -> PaymentRead: ...
```

## Кэширование и контент

- `Cache-Control`, `ETag`, `Last-Modified`; условные запросы `If-None-Match` → `304`.
- `Accept` учитывать; неподдерживаемое → `406`.
- Ошибки — `application/problem+json` (см. `errors.md`).

## Антипаттерны

| ❌                                           | ✅                        |
| -------------------------------------------- | ------------------------- |
| Глаголы в URI                                | существительные + методы  |
| Глубокая вложенность                         | top-level + фильтр        |
| Фильтры в пути                               | query-параметры           |
| `POST` для идемпотентного действия без ключа | `Idempotency-Key`         |
| Bulk «всё-или-ничего»                        | поэлементные результаты   |
| Разный casing                                | один стиль (`snake_case`) |

## Чек-лист

- [ ] Ресурсы — существительные, вложенность ≤ 2–3.
- [ ] Методы/идемпотентность корректны; `201`+`Location`, `204` на delete.
- [ ] Фильтры/сортировка/поиск — query, поля whitelisted.
- [ ] Bulk — поэлементно и с лимитом.
- [ ] `Idempotency-Key` на небезопасных POST.
- [ ] Кэш/условные заголовки, корректные content types.
