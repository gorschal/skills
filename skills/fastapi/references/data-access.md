# Доступ к данным (read-only)

Политика монорепозитория: **схемой и миграциями владеет Django**. FastAPI —
read-only потребитель: не создаёт таблицы, не пишет миграции, не коммитит.

## Что запрещено

- `Base.metadata.create_all()` — таблицы создаёт Django.
- Свои миграции в FastAPI (схема принадлежит Django).
- `session.commit()`/`rollback()` в репозитории.
- Запись в БД без согласованного контракта с владельцем.

## Сессия

```python
from collections.abc import AsyncGenerator
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker, create_async_engine

engine = create_async_engine(settings.database_url, pool_pre_ping=True)
async_session_maker = async_sessionmaker(engine, expire_on_commit=False)

async def get_db() -> AsyncGenerator[AsyncSession, None]:
    async with async_session_maker() as session:
        yield session
```

Драйвер — `asyncpg`. Сессия — через `Depends`, закрывается автоматически.

## Привязка к существующим таблицам

Два подхода (выбрать политикой проекта, зафиксировать ADR):

**A. Явные read-only ORM-модели** (рекомендуется для типизации):

```python
from sqlalchemy.orm import Mapped, mapped_column

class Payment(Base):
    __tablename__ = "payments"          # таблица существует, создана Django
    __table_args__ = {"extend_existing": True}

    id: Mapped[UUID] = mapped_column(primary_key=True)
    account_id: Mapped[UUID]
    amount: Mapped[int]
```

- Модель описывает только читаемые поля.
- Никакого `create_all`/своих миграций — Django остаётся источником истины.
- Дрейф схемы ловится контрактным тестом (сверка колонок с БД).

**B. `automap_base`** — рефлексия схемы без описания колонок:

```python
from sqlalchemy.ext.automap import automap_base

Base = automap_base()
async with engine.connect() as conn:
    await conn.run_sync(Base.prepare, reflect=True)
Payment = Base.classes.payments
```

Подходит для больших/волатильных схем, но даёт слабую типизацию. Использовать
осознанно.

## Репозиторий

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

Правила:

- Только SQLAlchemy (ORM/Core). Никакой бизнес-логики.
- Возврат: `None` / `TypedDict | None` / Pydantic / `list[...]`.
- Не возвращать `Row` наружу — маппить в `TypedDict`/Pydantic.
- Не вызывать `commit()`/`rollback()`.

## Eager loading (async)

Ленивая загрузка в async запрещена: она даёт неожиданные запросы и ошибки.

```python
from sqlalchemy.orm import selectinload, joinedload

stmt = (
    select(Order)
    .options(selectinload(Order.items))
    .where(Order.user_id == user_id)
)
```

- `selectinload` — коллекции (M2M, reverse FK).
- `joinedload` — FK/OneToOne.
- `lazy="raise"` на моделях — страховка от случайной ленивой загрузки.
- Никаких обращений к связи в цикле без предзагрузки (N+1).

## Сложные чтения через Core

Аналитика/агрегации — на стороне БД через `select()`:

```python
stmt = (
    select(Payment.account_id, func.sum(Payment.amount).label("total"))
    .where(Payment.created_at >= since)
    .group_by(Payment.account_id)
)
rows = (await session.execute(stmt)).all()
```

## Запись (если согласована)

По умолчанию FastAPI не пишет. Если запись необходима:

- Контракт с владельцем схемы (Django) согласован; инварианты не разъезжаются.
- Транзакция — в сервисе, не в репозитории.
- Критичные операции — идемпотентны.

```python
class PaymentService:
    def __init__(self, session: AsyncSession, repo: PaymentRepository) -> None:
        self.session = session
        self.repo = repo

    async def mark_processed(self, payment_id: UUID) -> None:
        async with self.session.begin():
            await self.repo.set_processed(payment_id)
```

## Антипаттерны

| ❌ | ✅ |
|---|---|
| `create_all`/свои миграции в FastAPI | Схема и миграции — Django |
| `commit()` в репозитории | Транзакция в сервисе (если согласована запись) |
| Ленивая загрузка в async | `selectinload`/`joinedload`, `lazy="raise"` |
| `Row` наружу | `TypedDict`/Pydantic |
| Синхронный драйвер | `asyncpg` |
| Бизнес-логика в репозитории | Сервис |
| Дублирование всех колонок вручную без проверки | Контрактный тест / automap |

## Чек-лист

- [ ] FastAPI не создаёт таблицы и не пишет миграции.
- [ ] Сессия через `Depends` (`asyncpg`), закрывается автоматически.
- [ ] ORM-модели описывают существующие таблицы; дрейф ловится тестом.
- [ ] Eager loading везде, где есть связи.
- [ ] Сложные чтения — через Core.
- [ ] Репозиторий не коммитит и не содержит бизнес-логику.
- [ ] Запись (если есть) согласована, транзакция — в сервисе.
