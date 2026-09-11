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
  version: "1.0.0"
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

> Общие практики языка — в навыке `python`. Схемой БД владеет Django (навык
> `django`); FastAPI — **read-only потребитель**.

## Когда применять

- Роутеры, зависимости, сервисы, репозитории, Pydantic-схемы.
- Чтение данных из БД, которой владеет Django.
- Обработка ошибок, JWT-аутентификация, CORS, OpenAPI/ReDoc.
- Unit-тесты сервисов.

## Ключевые принципы

1. **4 слоя:** Router → Service → Repository → Schemas. Границы неприкосновенны.
2. **Router тонкий**: HTTP-параметры, `Depends()`, один вызов сервиса.
3. **Бизнес-логика — в сервисе**; `HTTPException` в сервисе запрещён.
4. **Репозиторий — только доступ к данным**, без `commit`/`rollback` и бизнес-логики.
5. **Read-only:** схемой и миграциями владеет Django. FastAPI не создаёт таблицы
   и не пишет миграции.
6. **DI через `Depends`**; ручное создание сервисов запрещено.
7. **Pydantic v2:** `extra="forbid"` на входе, `from_attributes=True` на выходе.
8. **Ошибки — доменными исключениями** + глобальные handlers.
9. **Тесты:** BDD — основное покрытие; unit — логика вне BDD, зависимости `AsyncMock`.
10. **Docstring эндпоинтов — English** (попадает в ReDoc/OpenAPI).

## Архитектура (4 слоя)

| Слой | Файл | Разрешено |
|---|---|---|
| Router | `*_router.py` | HTTP-параметры, `Depends()`, один вызов сервиса |
| Service | `*_service.py` | Бизнес-логика, транзакции, доменные исключения |
| Repository | `*_repository.py` | Только запросы к БД (SQLAlchemy), без commit |
| Schemas | `*_schemas.py` | Только Pydantic-модели |

### Repository

```python
from typing import TypedDict
from uuid import UUID
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

class PaymentRow(TypedDict):
    id: UUID
    account_id: UUID
    amount: int

class PaymentRepository:
    def __init__(self, session: AsyncSession) -> None:
        self.session = session

    async def get_with_account(self, payment_id: UUID) -> PaymentRow | None:
        query = (
            select(Payment.id, Payment.account_id, Payment.amount)
            .join(Account, Payment.account_id == Account.id)
            .where(Payment.id == payment_id)
        )
        row = (await self.session.execute(query)).first()
        return PaymentRow(**row._mapping) if row else None
```

- Только SQLAlchemy (ORM поверх существующих таблиц или Core).
- Возврат: `None` / `TypedDict | None` / Pydantic / `list[...]`.
- ❌ `commit()`/`rollback()`, бизнес-логика, ORM-логика в сервисе.
- ❌ `create_all`, Alembic, свои миграции — схема принадлежит Django.

### Service

```python
class PaymentService:
    def __init__(self, repo: PaymentRepository) -> None:
        self.repo = repo

    async def get_payment(self, payment_id: UUID) -> PaymentOut:
        payment = await self.repo.get_with_account(payment_id)
        if payment is None:
            raise PaymentNotFoundError(payment_id)
        return PaymentOut.model_validate(payment)
```

- Зависимости — через `__init__`; роутер получает сервис через `Depends`.
- Сервис не знает про `Request`/`Response`; выбрасывает доменные исключения.

### Router

```python
from fastapi import APIRouter, Depends, status

router = APIRouter(prefix="/payments", tags=["payments"])

@router.get("/{payment_id}", response_model=PaymentOut)
async def get_payment(
    payment_id: UUID,
    service: PaymentService = Depends(get_payment_service),
) -> PaymentOut:
    """Return a single payment with its account.

    Raises:
        404: Payment not found.
    """
    return await service.get_payment(payment_id)
```

Подробно: [references/architecture.md](references/architecture.md).

## Dependency Injection

```python
from collections.abc import AsyncGenerator
from fastapi import Depends

async def get_db() -> AsyncGenerator[AsyncSession, None]:
    async with async_session_maker() as session:
        yield session

def get_payment_repository(db: AsyncSession = Depends(get_db)) -> PaymentRepository:
    return PaymentRepository(db)

def get_payment_service(
    repo: PaymentRepository = Depends(get_payment_repository),
) -> PaymentService:
    return PaymentService(repo)
```

- Ресурсы с cleanup — через `yield`; сервисы/репозитории — обычные фабрики.
- ❌ `service = PaymentService(db)` в роутере — только `Depends`.
- В тестах зависимости подменяются `app.dependency_overrides`.

## Доступ к данным (read-only)

- **Схема и миграции — Django.** FastAPI не создаёт таблицы и не пишет миграции.
- **ORM поверх существующих таблиц** для типизированного чтения; **Core**
  (`select()`) — для сложных/аналитических запросов.
- Без `Base.metadata.create_all()` и Alembic. Дрейф схемы отслеживается
  контрактным тестом.
- Eager loading обязателен: `selectinload`/`joinedload` или `lazy="raise"` —
  никаких ленивых запросов в async.
- Запись: только через согласованный контракт с Django (API/события); транзакция
  в таком случае — в сервисе, `commit` — не в репозитории.

```python
# сложное чтение через Core
stmt = (
    select(Payment.id, func.sum(Payment.amount).label("total"))
    .where(Payment.account_id == account_id)
    .group_by(Payment.id)
)
```

Подробно: [references/data-access.md](references/data-access.md).

## Pydantic v2

```python
class PaymentCreate(BaseModel):
    account_id: UUID
    amount: int = Field(gt=0)
    model_config = ConfigDict(extra="forbid")

class PaymentOut(BaseModel):
    id: UUID
    account_id: UUID
    amount: int
    model_config = ConfigDict(from_attributes=True)

class PaymentUpdate(BaseModel):
    amount: int | None = Field(default=None, gt=0)
    model_config = ConfigDict(extra="forbid")
```

- Три схемы: Create / Out / Update. `extra="forbid"` на входе.
- `from_attributes=True` — только для выходных/ORM-схем.
- `Annotated` для повторяющихся ограничений; `field_validator`/`model_validator`.
- ❌ `class Config`, `orm_mode = True`, `.dict()`, `parse_obj()` — это Pydantic v1.
- ❌ изменяемые значения по умолчанию → `Field(default_factory=...)`.

Подробно: [references/pydantic-v2.md](references/pydantic-v2.md).

## Асинхронность

| ❌ Запрещено | ✅ Замена |
|---|---|
| `requests.get()` | `httpx.AsyncClient` |
| `time.sleep()` | `await asyncio.sleep()` |
| `psycopg2`, `sqlite3` | `asyncpg` + SQLAlchemy async |
| `open()` | `aiofiles.open()` |
| блокирующий SDK | `run_in_threadpool` |

- `async def` — если есть `await`; иначе обычный `def` (FastAPI уведёт в threadpool).
- На все внешние вызовы — таймауты (`httpx`/`asyncio.timeout`).
- Тяжёлая работа — не в `BackgroundTasks`, а в очереди/Django Tasks.

## Обработка ошибок

```python
class DomainError(Exception): ...
class PaymentNotFoundError(DomainError): ...

@app.exception_handler(DomainError)
async def domain_error_handler(request: Request, exc: DomainError) -> JSONResponse:
    status = {PaymentNotFoundError: 404}.get(type(exc), 400)
    return JSONResponse(status_code=status, content={"code": type(exc).__name__, "detail": str(exc)})

@app.exception_handler(Exception)
async def unhandled_handler(request: Request, exc: Exception) -> JSONResponse:
    logger.exception("unhandled_error")
    return JSONResponse(status_code=500, content={"code": "internal_error", "detail": "Internal Server Error"})
```

- В сервисе — только доменные исключения; `HTTPException` запрещён.
- Глобальные handlers: `DomainError`, `StarletteHTTPException`,
  `RequestValidationError`, `Exception`.
- Стектрейс и детали — только в логи; клиенту — безопасное сообщение.

Подробно: [references/errors.md](references/errors.md).

## Конфигурация и логирование

```python
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    database_url: str
    secret_key: SecretStr
    debug: bool = False
    model_config = SettingsConfigDict(env_file=".env")

settings = Settings()
```

- `os.getenv()` вне `config.py` запрещён; секреты — из окружения.
- structlog: события `snake_case`, параметры `key=value`, без `print`/f-строк.
- `request_id` — middleware + `structlog.contextvars`, очищать в `finally`.

## Тестирование

Политика: **BDD — основное сквозное покрытие**; unit — логика вне BDD и
критичные ветки. Unit не дублирует BDD. HTTP-слой проверяет BDD.

```python
from unittest.mock import AsyncMock
import pytest

async def test_get_payment_raises_when_missing() -> None:
    repo = AsyncMock()
    repo.get_with_account.return_value = None
    service = PaymentService(repo=repo)

    with pytest.raises(PaymentNotFoundError):
        await service.get_payment(UUID(int=1))

    repo.get_with_account.assert_awaited_once()
```

- pytest-функции и фикстуры; зависимости — `AsyncMock`.
- **Запрещено в unit:** `TestClient`, `httpx.AsyncClient` по приложению, реальная БД.
- Тестируем: бизнес-логику сервисов, маппинг ошибок, ветвления. Не тестируем:
  Pydantic-валидацию, простой CRUD, HTTP-статусы (BDD).

Подробно: [references/testing.md](references/testing.md).

## Документирование

- Docstring эндпоинтов — **English**, это `description` в OpenAPI/ReDoc.
- `summary` — из имени функции или `summary=`; указывать коды и ошибки.
- Сервисы — «почему» + ограничения; репозитории — смысл запроса.
- Не документировать `__init__`, простой CRUD.

```python
@router.get("/{payment_id}", response_model=PaymentOut)
async def get_payment(...) -> PaymentOut:
    """Return a single payment with its account.

    Raises:
        404: Payment not found.
    """
```

Общая матрица — в навыке `python`, `references/documentation.md`.

## Безопасность

- Пароли — `argon2`/`bcrypt`, не в открытом виде.
- JWT: короткий срок жизни, алгоритм зафиксирован, секрет из env.
- `OAuth2PasswordBearer` + `get_current_user`; права проверять на каждом роуте.
- CORS — только разрешённые origin; не `*` с credentials.
- Docs (`/docs`, `/openapi.json`) отключать в проде.
- Валидация входа (Pydantic), параметризованный SQL, секреты не в логах.

Подробно: [references/security.md](references/security.md).

## Инструменты

```bash
docker compose exec fastapi uv run ruff check . && docker compose exec fastapi uv run ruff format --check .
docker compose exec fastapi uv run pyright
docker compose exec fastapi uv run pytest -v
```

## Запрещённые паттерны

| ❌ Запрещено | ✅ Правильно |
|---|---|
| SQL/бизнес-логика в роутере | SQL в repository, логика в service |
| `service = PaymentService(db)` в роутере | `service: PaymentService = Depends(...)` |
| `HTTPException` в сервисе | Доменное исключение + handler |
| `commit()`/`rollback()` в репозитории | Транзакция в сервисе (если запись согласована) |
| `create_all`/Alembic/свои миграции | Схема и миграции — Django |
| Ленивая загрузка связей в async | `selectinload`/`joinedload`, `lazy="raise"` |
| `class Config` / `orm_mode` / `.dict()` | `ConfigDict` / `from_attributes` / `model_dump()` |
| `requests`/`time.sleep`/sync-драйверы | `httpx`/`asyncio.sleep`/`asyncpg` |
| `print()` / f-строки в логах | `logger.info("event", key=value)` |
| `TestClient` в unit-тестах | `AsyncMock` зависимостей; HTTP — BDD |
| `os.getenv()` вне config | `pydantic-settings` |
| Стектрейс/детали в ответе | Общий handler + лог |

## Чек-лист code review

- [ ] Router тонкий, один вызов сервиса, зависимости через `Depends`.
- [ ] Бизнес-логика в сервисе; сервис не знает про `Request`.
- [ ] Репозиторий — только SQLAlchemy; без `commit`/`rollback`.
- [ ] Схема БД не дублируется: нет `create_all`/миграций в FastAPI.
- [ ] Связи загружаются явно (нет ленивых запросов/N+1).
- [ ] Pydantic v2: `extra="forbid"` на входе, `from_attributes` на выходе.
- [ ] Доменные исключения; глобальные handlers зарегистрированы.
- [ ] Нет блокирующих вызовов в async; таймауты заданы.
- [ ] Секреты из env; не логируются.
- [ ] Docstring эндпоинтов на английском, пригодны для ReDoc.
- [ ] Unit-тесты не дублируют BDD; зависимости `AsyncMock`; нет `TestClient`.
- [ ] `ruff check`, `ruff format --check`, `pyright`, тесты проходят.

## Справочники

| Тема | Reference | Загружать когда |
|---|---|---|
| Архитектура, слои, DI, lifespan | [references/architecture.md](references/architecture.md) | Проектирование роутеров/сервисов/структуры |
| Доступ к данным | [references/data-access.md](references/data-access.md) | Сессии, ORM/Core, read-only, eager loading |
| Pydantic v2 | [references/pydantic-v2.md](references/pydantic-v2.md) | Схемы, валидаторы, настройки |
| Ошибки | [references/errors.md](references/errors.md) | Доменные исключения, global handlers, middleware |
| Безопасность | [references/security.md](references/security.md) | JWT, OAuth2, CORS, docs, hashing |
| Тестирование | [references/testing.md](references/testing.md) | unit-тесты сервисов, моки, границы BDD |

## Связанные навыки

- `python` — общие практики языка.
- `django` — владелец схемы и миграций.
- `python-testing` / `pytest-bdd` — тестовая инфраструктура и BDD.
- `security` — расширенный чек-лист безопасности.
- `api-design` — REST, версионирование, пагинация, OpenAPI.
- `postgres` — индексы, планы, производительность БД.
