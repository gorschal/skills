# Архитектура FastAPI-приложения

Тонкие роутеры, изолированный сервисный слой, явные зависимости, предсказуемый
старт/стоп. FastAPI — read-only потребитель схемы, которой владеет Django.

## Слои и границы

| Слой | Знает про | Не знает про | Тестируется |
|---|---|---|---|
| Router | HTTP, `Depends`, схемы | SQL, бизнес-логику | BDD |
| Service | домен, репозитории, транзакции | `Request`/`Response` | unit (моки) |
| Repository | БД, SQLAlchemy | бизнес-правила, commit | через сервис/BDD |
| Schemas | форму и валидацию данных | персистентность | через BDD |

Правило: **один вызов сервиса из роутера**; сервис оркестрирует репозитории.

## Структура проекта

```
app/
├── main.py            # FastAPI(...), lifespan, include_router, exception handlers
├── config.py          # Settings (pydantic-settings)
├── db.py              # engine, async_sessionmaker, get_db()
├── errors.py          # доменные исключения + обработчики
├── logging.py         # structlog
├── api/
│   └── v1/
│       ├── deps.py            # DI-фабрики
│       └── payments/
│           ├── router.py
│           ├── service.py
│           ├── repository.py
│           └── schemas.py
└── tests/
```

## APIRouter и версионирование

```python
# app/api/v1/payments/router.py
router = APIRouter(prefix="/payments", tags=["payments"])

# app/main.py
app.include_router(payments.router, prefix="/api/v1")
```

`prefix` и `tags` — на роутере, а не в каждой ручке. Версия — в префиксе.

## lifespan

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI

@asynccontextmanager
async def lifespan(app: FastAPI):
    engine = create_async_engine(settings.database_url, pool_pre_ping=True)
    app.state.sessionmaker = async_sessionmaker(engine, expire_on_commit=False)
    app.state.http = httpx.AsyncClient(timeout=10.0)
    yield
    await app.state.http.aclose()
    await engine.dispose()

app = FastAPI(lifespan=lifespan)
```

- Ресурсы — в `app.state`, доступ через `Depends`, не через глобальные переменные.
- `engine.dispose()` и закрытие клиентов на shutdown обязательны.
- `@app.on_event` устарел — использовать `lifespan`.

## Dependency Injection

```python
from collections.abc import AsyncGenerator
from fastapi import Depends

# Ресурс с cleanup — yield
async def get_db() -> AsyncGenerator[AsyncSession, None]:
    async with async_session_maker() as session:
        yield session

# Фабрика репозитория
def get_payment_repository(db: AsyncSession = Depends(get_db)) -> PaymentRepository:
    return PaymentRepository(db)

# Фабрика сервиса
def get_payment_service(
    repo: PaymentRepository = Depends(get_payment_repository),
) -> PaymentService:
    return PaymentService(repo)
```

- Ресурсы (БД, HTTP-клиент) — `yield`; сервисы/репозитории — обычные функции.
- ❌ Ручное создание в роутере: `service = PaymentService(db)`.
- Тесты подменяют зависимости через `app.dependency_overrides`.

## Сервис

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

- Зависимости — через `__init__`; никаких глобальных синглтонов.
- Сервис не знает про HTTP; принимает/возвращает данные и схемы.
- Доменные исключения — здесь.

## Роутер

```python
router = APIRouter(prefix="/payments", tags=["payments"])

@router.get("/{payment_id}", response_model=PaymentOut)
async def get_payment(
    payment_id: UUID,
    service: PaymentService = Depends(get_payment_service),
) -> PaymentOut:
    """Return a single payment with its account."""
    return await service.get_payment(payment_id)
```

- Роутер: параметры → `Depends` → один вызов → `response_model`.
- Без SQL, без бизнес-логики, без `try/except` бизнес-ошибок.

## Антипаттерны

| ❌ | ✅ |
|---|---|
| SQL/логика в роутере | Repository/Service |
| `service = Service(db)` в роутере | `Depends(get_..._service)` |
| Глобальные синглтоны | `app.state` + `Depends` |
| `@app.on_event` | `lifespan` |
| Смешение ORM-моделей и API-схем | Отдельные `models`/`schemas` |
| Версия в каждой ручке | Префикс роутера |

## Чек-лист

- [ ] 4 слоя, границы соблюдены.
- [ ] Роутеры тонкие, зависимости через `Depends`.
- [ ] Сервис не знает про `Request`/`Response`.
- [ ] Ресурсы в `lifespan`/`app.state`, не глобальные.
- [ ] Версионирование через префикс.
- [ ] ORM-модели и Pydantic-схемы — разные классы.
