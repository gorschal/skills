---
name: api-design
description: >
  Use when designing or reviewing HTTP APIs: resource modeling, HTTP methods and
  status codes, pagination, error responses (RFC 7807), versioning/deprecation,
  and OpenAPI. Триггеры: API design, REST, OpenAPI, endpoint, pagination,
  cursor, versioning, status codes, RFC 7807, problem+json, idempotency key,
  resource modeling, contract. Реализация на FastAPI — навык fastapi.
license: MIT
compatibility: opencode
metadata:
  version: "1.0.0"
  domain: api-architecture
  triggers: API design, REST, OpenAPI, pagination, versioning, status codes, RFC 7807, idempotency, resource modeling
  role: architect
  scope: design
  output-format: specification
  related-skills: fastapi, django, security
---

# API Design

Проектирование HTTP API: ресурсы, методы, пагинация, ошибки, версионирование,
OpenAPI. Реализация на FastAPI — в навыке `fastapi`; здесь — контракт и принципы.

## Когда применять

- Моделирование ресурсов, эндпоинтов, контрактов.
- Пагинация, ошибки, версионирование, OpenAPI.
- Ревью API на консистентность и обратную совместимость.

## Принципы

1. **Ресурсы — существительные**, действия — HTTP-методы.
2. **Один стиль именования** во всём API (для Python — `snake_case`).
3. **Коллекции всегда пагинируются.**
4. **Единый формат ошибки** (RFC 7807) и корректные статус-коды.
5. **Версия с первого дня** (`/v1`), ломающие изменения — только в новой версии.
6. **OpenAPI генерируется из кода** и не расходится с реализацией.
7. **Не раскрывать внутренности** (стектрейсы, SQL, пути).

## Ресурсы и методы

```
GET    /v1/users                 # список
POST   /v1/users                 # создание
GET    /v1/users/{id}            # чтение
PUT    /v1/users/{id}            # полная замена
PATCH  /v1/users/{id}            # частичное обновление
DELETE /v1/users/{id}            # удаление
GET    /v1/users/{id}/orders     # вложенная коллекция (не глубже 2–3 уровней)
```

| Метод | Safe | Idempotent | Назначение |
|---|---|---|---|
| GET | да | да | чтение |
| POST | нет | нет | создание/действие |
| PUT | нет | да | полная замена |
| PATCH | нет | нет | частичное обновление |
| DELETE | нет | да | удаление |

- Фильтры/сортировка/поиск — **query-параметры**, не путь:
  `GET /v1/users?status=active&sort=-created_at&q=john&limit=20`.
- Вложенность ≤ 2–3 уровней; глубже — отдельный ресурс с фильтром.
- `POST` не идемпотентен → для платежей/созданий — `Idempotency-Key`.
- PUT заменяет целиком; PATCH трогает только переданные поля.

### Статус-коды

`200` OK, `201` Created (+`Location`), `202` Accepted, `204` No Content;
`400`/`422` валидация, `401` не аутентифицирован, `403` запрещено, `404` нет,
`409` конфликт, `429` rate limit (+`Retry-After`); `500`/`503`.

Подробно: [references/rest.md](references/rest.md).

## Пагинация

- Всегда пагинировать коллекции; default 20–50, max 100–1000.
- Маленькие/стабильные данные — offset/page; большие/меняющиеся — **cursor/keyset**.
- Один паттерн во всём API; envelope с `data` + `pagination` + `links`.
- Cursor — непрозрачный, кодирует ключи сортировки + уникальный tiebreaker.
- Стабильный `ORDER BY`, заканчивающийся уникальной колонкой.

```json
{
  "data": [ ... ],
  "pagination": { "limit": 20, "next_cursor": "...", "has_more": true },
  "links": { "self": "...", "next": "..." }
}
```

Порядок обработки: **filter → count → sort → paginate**.

Подробно: [references/pagination.md](references/pagination.md).

## Ошибки (RFC 7807)

Единый формат — `application/problem+json`:

```json
{
  "type": "https://api.example.com/errors/resource-not-found",
  "title": "Resource Not Found",
  "status": 404,
  "detail": "User 123 does not exist",
  "instance": "/v1/users/123",
  "code": "RESOURCE_NOT_FOUND",
  "request_id": "req_abc123"
}
```

- `type` — стабильный URI класса ошибки; `code` — машинный код (расширение).
- `errors[]` — детали валидации по полям.
- `X-Request-ID` в заголовке и теле.
- `500` — общее сообщение; детали только в логах.
- `RequestValidationError` маппится в тот же envelope.

Подробно: [references/errors.md](references/errors.md).

## Версионирование

- URI-версия (`/v1`) — самый явный вариант.
- Только мажорные версии; ломающие изменения → новая версия.
- Не ломающие: новые эндпоинты/опциональные поля/поля ответа.
- Ломающие: удаление/переименование полей, смена типа, новый обязательный
  параметр, смена статус-кода/аутентификации.
- Deprecation: заголовки `Deprecation`, `Sunset`, `Link rel="successor-version"`;
  после sunset — `410 Gone`.
- Держать ≤ 3 активных версий; окно депрекации ≥ 6 месяцев.

Подробно: [references/versioning.md](references/versioning.md).

## OpenAPI

- Спека генерируется из кода (FastAPI) — единственный источник правды.
- Каждая операция: уникальный `operationId`, summary, description, tags,
  **все** ответы (включая ошибки).
- Переиспользовать компоненты (`$ref`), добавлять примеры.
- Security schemes объявлены и применены.
- Валидация/линт спеки в CI (`swagger-cli`, Spectral).

```python
@router.get("/users", response_model=Page[UserRead], summary="List users",
            operation_id="listUsers", tags=["Users"],
            responses={401: {"$ref": "#/components/responses/Unauthorized"}})
async def list_users(...): ...
```

Подробно: [references/openapi.md](references/openapi.md).

## Запрещённые паттерны

| ❌ Запрещено | ✅ Правильно |
|---|---|
| Глаголы в URI (`/getUser`) | `/users/{id}` |
| Несогласованные структуры ответа | единый envelope |
| `200` с телом ошибки | корректный статус-код |
| Коллекция без пагинации | всегда пагинировать |
| Большой `OFFSET` | cursor/keyset |
| `total` в cursor-пагинации | `has_more`/`next_cursor` |
| Разный формат ошибок | RFC 7807 + `code` |
| Стектрейс/SQL в ответе | общее сообщение + `request_id` |
| Смена типа/удаление поля без версии | новая версия |
| Версия не с первого дня | `/v1` сразу |
| Устаревшая спека | генерация из кода + CI |
| Незадокументированные ошибки | все ответы в OpenAPI |

## Чек-лист

- [ ] Ресурсы — существительные, без глаголов; вложенность ≤ 2–3.
- [ ] Методы и идемпотентность корректны; `201`+`Location`, `204` на delete.
- [ ] Коллекции пагинируются; default/max заданы и задокументированы.
- [ ] Пагинация консистентна; курсор непрозрачный, сортировка стабильна.
- [ ] Единый формат ошибки (RFC 7807) с `code`/`request_id`.
- [ ] Все статус-коды и коды ошибок задокументированы.
- [ ] Версия `/v1`; ломающие изменения — новая версия; deprecation-заголовки.
- [ ] OpenAPI генерируется из кода и проходит линт в CI.
- [ ] Именование консистентно (`snake_case`); внутренности не раскрываются.

## Справочники

| Тема | Reference | Загружать когда |
|---|---|---|
| REST-паттерны | [references/rest.md](references/rest.md) | Ресурсы, методы, статус-коды, bulk |
| Пагинация | [references/pagination.md](references/pagination.md) | Offset/cursor, envelope, лимиты |
| Ошибки | [references/errors.md](references/errors.md) | RFC 7807, статус-коды, валидация |
| Версионирование | [references/versioning.md](references/versioning.md) | Версии, deprecation, breaking changes |
| OpenAPI | [references/openapi.md](references/openapi.md) | Спека, схемы, теги, security |

## Связанные навыки

- `fastapi` — реализация API (роутеры, Pydantic, handlers).
- `security` — аутентификация/авторизация, rate limiting.
- `django` — server-rendered часть; API — на FastAPI.
