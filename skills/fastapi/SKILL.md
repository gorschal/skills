---
name: fastapi
description: >
  Use when building FastAPI 0.115+ async APIs: APIRouter, dependency injection
  with Depends, Pydantic v2 schemas, read-only SQLAlchemy access to Django-owned
  tables, domain errors and global handlers, JWT security, structlog, unit tests.
  Триггеры: FastAPI, APIRouter, Depends, Pydantic v2, SQLAlchemy async,
  asyncpg, httpx, structlog, OpenAPI, ReDoc, lifespan.
  Общие практики Python — навык python; владелец схемы БД — навык django.
license: MIT
compatibility: opencode
metadata:
  version: "1.1.0"
  domain: backend
  triggers: FastAPI, APIRouter, Depends, Pydantic, SQLAlchemy, asyncpg, httpx, OpenAPI, lifespan
  role: specialist
  scope: implementation
  output-format: code
  related-skills: python, python-testing, pytest-bdd, django, security, api-design, postgres
---

# FastAPI

FastAPI 0.115+ для async API: 4 слоя, DI через `Depends`, Pydantic v2,
read-only доступ к таблицам, которыми владеет Django.

> Общие практики языка — `python`. Схемой БД владеет Django (`django`); FastAPI —
> **read-only потребитель**.

## Когда применять

- Роутеры, DI, сервисы, репозитории, Pydantic-схемы.
- Чтение данных из БД Django; ошибки, JWT, CORS, OpenAPI/ReDoc; unit-тесты.

## Ключевые принципы

1. **4 слоя:** Router → Service → Repository → Schemas. Границы неприкосновенны.
2. **Router тонкий**: параметры, `Depends()`, один вызов сервиса.
3. **Бизнес-логика — в сервисе**; `HTTPException` в сервисе запрещён.
4. **Репозиторий — только доступ к данным**, без `commit`/`rollback`.
5. **Read-only:** схема/миграции — Django; FastAPI не создаёт таблицы.
6. **DI через `Depends`**; ручное создание сервисов запрещено.
7. **Pydantic v2:** `extra="forbid"` на входе, `from_attributes=True` на выходе.
8. **Ошибки — доменными исключениями** + глобальные handlers (RFC 7807).
9. **Тесты:** BDD — покрытие; unit — пробелы, зависимости `AsyncMock`.
10. **Docstring эндпоинтов — English** (ReDoc/OpenAPI).

## Архитектура (4 слоя)

| Слой       | Файл              | Разрешено                                       |
| ---------- | ----------------- | ----------------------------------------------- |
| Router     | `*_router.py`     | HTTP-параметры, `Depends()`, один вызов сервиса |
| Service    | `*_service.py`    | бизнес-логика, транзакции, доменные исключения  |
| Repository | `*_repository.py` | только запросы к БД (SQLAlchemy), без commit    |
| Schemas    | `*_schemas.py`    | только Pydantic-модели                          |

Псевдо-шаблон (детали — в references):

```python
# repository: select(...) → TypedDict/Pydantic; без commit
# service: repo через __init__; raise DomainError; return Out.model_validate(...)
# router: @router.get(..., response_model=Out)
async def get_payment(payment_id: UUID, service: PaymentService = Depends(get_payment_service)) -> PaymentOut:
    return await service.get_payment(payment_id)
```

- Только SQLAlchemy (ORM поверх существующих таблиц / Core); возврат `None`/`TypedDict`/Pydantic/`list`.
- ❌ `commit`/`rollback`, бизнес-логика в репозитории, `create_all`/свои миграции (схема — Django).

Подробно: [references/architecture.md](references/architecture.md).

## Dependency Injection

```python
async def get_db() -> AsyncGenerator[AsyncSession, None]:
    async with async_session_maker() as session:
        yield session

def get_payment_service(repo: PaymentRepository = Depends(get_payment_repository)) -> PaymentService:
    return PaymentService(repo)
```

- Ресурсы — `yield`; сервисы/репозитории — обычные фабрики.
- ❌ `PaymentService(db)` в роутере; в тестах — `app.dependency_overrides`.

## Доступ к данным (read-only)

- **Схема и миграции — Django**; без `create_all`/своих миграций; дрейф — контрактным тестом.
- **ORM поверх существующих таблиц**; **Core** — для сложных/аналитических запросов.
- Eager loading обязателен: `selectinload`/`joinedload`/`lazy="raise"`.
- Запись — только по согласованному контракту с Django; транзакция в сервисе.

Подробно: [references/data-access.md](references/data-access.md).

## Pydantic v2

```python
class PaymentCreate(BaseModel):
    amount: int = Field(gt=0)
    model_config = ConfigDict(extra="forbid")

class PaymentOut(BaseModel):
    id: UUID
    model_config = ConfigDict(from_attributes=True)
```

- Три схемы: Create / Out / Update; `extra="forbid"` на входе.
- `from_attributes=True` — только для выходных; `Annotated`/`field_validator`.
- ❌ `class Config`, `orm_mode`, `.dict()`, `parse_obj()` (Pydantic v1); изменяемые дефолты.

Подробно: [references/pydantic-v2.md](references/pydantic-v2.md).

## Асинхронность

| ❌                   | ✅                           |
| -------------------- | ---------------------------- |
| `requests.get()`     | `httpx.AsyncClient`          |
| `time.sleep()`       | `await asyncio.sleep()`      |
| `psycopg2`/`sqlite3` | `asyncpg` + SQLAlchemy async |
| `open()`             | `aiofiles.open()`            |
| блокирующий SDK      | `run_in_threadpool`          |

- `async def` — если есть `await`; иначе `def` (FastAPI уведёт в threadpool).
- Таймауты на внешние вызовы; тяжёлое — в очередь/Django Tasks, не в `BackgroundTasks`.

## Обработка ошибок

Единый формат — **RFC 7807** (`application/problem+json`); контракт — `api-design`.

```python
@app.exception_handler(DomainError)
async def domain_error_handler(request: Request, exc: DomainError) -> JSONResponse:
    status = {PaymentNotFoundError: 404}.get(type(exc), 400)
    return problem(request, status=status, code=type(exc).__name__.upper(), detail=str(exc))
```

- В сервисе — только доменные исключения; `HTTPException` запрещён.
- Handlers: `DomainError`, `StarletteHTTPException`, `RequestValidationError`, `Exception`.
- Формат: `type/title/status/detail/instance` + `code`/`request_id`/`errors`.
- Стектрейс и детали — только в логи.

Подробно: [references/errors.md](references/errors.md).

## Конфигурация и логирование

- `pydantic-settings`; `os.getenv()` вне `config.py` запрещён; секреты из env.
- structlog: события `snake_case`, `key=value`, без `print`/f-строк.
- `request_id` — middleware + `contextvars`, очищать в `finally`.

## Тестирование

Политика: **BDD — покрытие**; unit — пробелы и критичные ветки; HTTP — BDD.

- pytest-функции/фикстуры; зависимости — `AsyncMock`.
- **Запрещено в unit:** `TestClient`, `httpx.AsyncClient` по приложению, реальная БД.
- Тестируем логику сервисов и маппинг ошибок; не тестируем Pydantic/CRUD/статусы.

Подробно: [references/testing.md](references/testing.md).

## Документирование

- Docstring эндпоинтов — **English** (`description` в OpenAPI/ReDoc); указывать коды/ошибки.
- Сервисы — «почему» + ограничения; репозитории — смысл запроса.
- Не документировать `__init__`, простой CRUD. Матрица — `python/references/documentation.md`.

## Безопасность

- Пароли — `argon2`/`bcrypt`; JWT — короткий срок, зафиксированный алгоритм, секрет из env.
- `OAuth2PasswordBearer` + `get_current_user`; права на каждом роуте.
- CORS — только разрешённые origin; docs отключать в проде.
- Валидация входа, параметризованный SQL, секреты не в логах.

Подробно: [references/security.md](references/security.md) и навык `security`.

## Инструменты

```bash
docker compose exec fastapi uv run ruff check . && docker compose exec fastapi uv run ruff format --check .
docker compose exec fastapi uv run pyright
docker compose exec fastapi uv run pytest -v
```

## Запрещённые паттерны

| ❌                                    | ✅                                            |
| ------------------------------------- | --------------------------------------------- |
| SQL/логика в роутере                  | repository/service                            |
| `PaymentService(db)` в роутере        | `Depends(...)`                                |
| `HTTPException` в сервисе             | доменное исключение + handler                 |
| `commit()` в репозитории              | транзакция в сервисе                          |
| `create_all`/свои миграции            | схема/миграции — Django                       |
| ленивая загрузка в async              | `selectinload`/`joinedload`, `lazy="raise"`   |
| `class Config`/`orm_mode`/`.dict()`   | `ConfigDict`/`from_attributes`/`model_dump()` |
| `requests`/`time.sleep`/sync-драйверы | `httpx`/`asyncio.sleep`/`asyncpg`             |
| `print()`/f-строки в логах            | `logger.info("event", key=value)`             |
| `TestClient` в unit                   | `AsyncMock`; HTTP — BDD                       |
| `os.getenv()` вне config              | `pydantic-settings`                           |
| стектрейс в ответе                    | общий handler + лог                           |

## Чек-лист code review

- [ ] Router тонкий, один вызов сервиса, зависимости через `Depends`.
- [ ] Бизнес-логика в сервисе; сервис не знает про `Request`.
- [ ] Репозиторий — только SQLAlchemy; без `commit`/`rollback`.
- [ ] Нет `create_all`/миграций в FastAPI.
- [ ] Связи загружаются явно (нет N+1).
- [ ] Pydantic v2: `extra="forbid"` на входе, `from_attributes` на выходе.
- [ ] Доменные исключения; handlers зарегистрированы (RFC 7807).
- [ ] Нет блокирующих вызовов в async; таймауты заданы.
- [ ] Секреты из env; не логируются.
- [ ] Docstring эндпоинтов на английском (ReDoc).
- [ ] Unit не дублирует BDD; нет `TestClient`; `ruff`/`pyright`/тесты проходят.

## Справочники

| Тема                            | Reference                                                | Когда                       |
| ------------------------------- | -------------------------------------------------------- | --------------------------- |
| Архитектура, слои, DI, lifespan | [references/architecture.md](references/architecture.md) | Проектирование              |
| Доступ к данным                 | [references/data-access.md](references/data-access.md)   | Сессии, ORM/Core, read-only |
| Pydantic v2                     | [references/pydantic-v2.md](references/pydantic-v2.md)   | Схемы, валидаторы           |
| Ошибки                          | [references/errors.md](references/errors.md)             | RFC 7807, handlers          |
| Безопасность                    | [references/security.md](references/security.md)         | JWT, CORS, docs, hashing    |
| Тестирование                    | [references/testing.md](references/testing.md)           | unit, моки, BDD             |

## Связанные навыки

- `python` — общие практики; `django` — владелец схемы.
- `python-testing`/`pytest-bdd` — тесты; `security` — безопасность.
- `api-design` — REST, версионирование, OpenAPI; `postgres` — производительность.
