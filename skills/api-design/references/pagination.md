# Пагинация

Все коллекции пагинируются. Default 20–50, max 100–1000; лимиты заданы и
задокументированы.

## Выбор стратегии

|                     | Offset/page                     | Cursor/keyset             |
| ------------------- | ------------------------------- | ------------------------- |
| Производительность  | плохая на больших offset        | отличная                  |
| Произвольный доступ | да                              | нет                       |
| `total`             | да                              | нет                       |
| Консистентность     | слабая                          | отличная                  |
| Для чего            | малые/стабильные наборы, веб-UI | большие/меняющиеся, ленты |

## Offset / page

```
GET /v1/users?offset=20&limit=10
GET /v1/users?page=3&per_page=10
```

Плюсы: простота, произвольный доступ, `total`. Минусы: медленный глубокий offset,
пропуски/дубли при изменениях, дорогой `COUNT`.

## Cursor / keyset

```
GET /v1/users?limit=10
GET /v1/users?cursor=eyJpZCI6MzB9&limit=10
GET /v1/users?after_id=20&limit=10
```

```sql
SELECT * FROM users
WHERE (created_at, id) < (:last_created_at, :last_id)
ORDER BY created_at DESC, id DESC
LIMIT :limit;
```

Плюсы: консистентность, индекс, без `COUNT`. Минусы: нет произвольного доступа и
`total`, курсор привязан к полям сортировки.

## Envelope

```json
{
  "data": [{ "id": 21, "name": "User 21" }],
  "pagination": {
    "limit": 10,
    "next_cursor": "eyJpZCI6MzB9",
    "has_more": true
  },
  "links": {
    "self": "/v1/users?limit=10",
    "next": "/v1/users?cursor=eyJpZCI6MzB9&limit=10"
  }
}
```

Offset-вариант: `offset`, `total`, `has_previous`, ссылки `first`/`prev`/`next`/`last`.

```python
from typing import Generic, TypeVar
from pydantic import BaseModel, Field

T = TypeVar("T")

class Page(BaseModel, Generic[T]):
    data: list[T]
    limit: int
    offset: int | None = None
    next_cursor: str | None = None
    total: int | None = None
    has_more: bool

class PaginationParams(BaseModel):
    limit: int = Field(20, ge=1, le=100)
    offset: int = Field(0, ge=0)
```

## Лимиты

```python
limit: Annotated[int, Query(ge=1, le=100, description="Items per page (max 100)")] = 20
```

Выход за диапазон → `422` с понятным сообщением; default/max задокументированы.

## `total`

- Включать для малых/стабильных наборов и page-UI.
- Опускать (использовать `has_more`) для больших/real-time и cursor.
- Дорогой `total` — опционально (`?include_total=true`).

## Стабильный порядок

- Детерминированный `ORDER BY`, заканчивающийся уникальной колонкой (`id`).
- Для мультиполей курсор кодирует все поля сортировки.
- Смена сортировки инвалидирует in-flight курсор.

## Filter + sort + paginate

Порядок: **filter → count → sort → paginate**. Фильтры/сортировку сохранять во
всех ссылках.

```
GET /v1/users?status=active&sort=-created_at&limit=10&offset=0
```

## Антипаттерны

| ❌                                 | ✅                           |
| ---------------------------------- | ---------------------------- |
| Коллекция без пагинации            | всегда пагинировать          |
| Глубокий `OFFSET`                  | cursor/keyset                |
| `total` в cursor-пагинации         | `has_more`/`next_cursor`     |
| Разные паттерны в API              | один паттерн                 |
| Непрозрачный курсор без sort-полей | кодировать sort + tiebreaker |
| Сортировка без tiebreaker          | + уникальная колонка         |

## Чек-лист

- [ ] Все коллекции пагинируются; default/max заданы.
- [ ] Стратегия выбрана под данные и консистентна.
- [ ] `has_more` всегда; `total` опционально.
- [ ] Ссылки сохраняют фильтры/сортировку.
- [ ] Детерминированный порядок с уникальным tiebreaker.
- [ ] Курсор непрозрачный и валидируется.
