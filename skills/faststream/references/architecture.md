# Архитектура FastStream-сервиса

Тонкий handler, изолированный сервис, доступ к данным в репозитории.
FastStream — тонкий клиент брокера: оркестрацию/ретраи он не делает.

## Слои и границы

| Слой | Знает про | Не знает про | Тестируется |
|---|---|---|---|
| Handler | сообщение, `Depends` | SQL, бизнес-логику | BDD/`TestNatsBroker` |
| Service | домен, репозитории, транзакции | NATS, subject, headers | unit (моки) |
| Repository | БД, SQLAlchemy | бизнес-правила, commit | через сервис/QA |
| Schemas | форму сообщений | персистентность | Pydantic (не unit) |

Правило: **один вызов сервиса из handler**; сервис оркестрирует репозитории.

## Структура проекта

```
src/
├── main.py              # FastStream(broker), подключение роутеров
├── broker.py            # NatsBroker, настройки
├── handlers/
│   └── payments.py      # @broker.subscriber
├── services/
├── repositories/
├── schemas/
├── core/
│   ├── config.py
│   ├── database.py
│   ├── exceptions.py
│   └── logger.py
└── tests/
```

## Handler

```python
@broker.subscriber("payments.process", queue="payments-workers")
async def process_payment_handler(
    message: PaymentProcessMessage,
    service: PaymentService = Depends(get_payment_service),
) -> None:
    await service.process(message)
```

- Принимает типизированное сообщение, вызывает сервис, ничего не возвращает
  (или возвращает результат для RPC).
- Без бизнес-логики, без доступа к БД и брокеру напрямую.
- Subject/queue — только здесь и в конфиге.

## Сервис

```python
class PaymentService:
    def __init__(self, session: AsyncSession, repo: PaymentRepository) -> None:
        self.session = session
        self.repo = repo

    async def process(self, message: PaymentProcessMessage) -> None:
        if await self.repo.is_processed(message.payment_id):
            raise PaymentAlreadyProcessedError(message.payment_id)
        async with self.session.begin():
            await self.repo.mark_processed(message.payment_id)
            await self.repo.apply(message)
```

- Зависимости — через `__init__`; сервис не знает про NATS.
- Идемпотентность и транзакции — здесь.
- Доменные исключения — здесь.

## Repository

- Только доступ к данным (SQLAlchemy Core/ORM).
- Не вызывает `commit()`/`rollback()`.
- Возвращает `TypedDict`/Pydantic/`None`/`list[...]`.

## Dependency Injection

```python
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

- Ресурсы — `yield`; сервисы/репозитории — обычные фабрики.
- ❌ Ручное создание в handler.

## Роутеры

Несколько доменов — через `NatsRouter`/`StreamRouter` и подключение к `FastStream`.

## Антипаттерны

| ❌ | ✅ |
|---|---|
| Бизнес-логика в handler | Service |
| SQL в handler | Repository |
| `service = PaymentService(db)` | `Depends(...)` |
| `commit()` в repository | Транзакция в сервисе |
| Хардкод subject в service | handler/config |
| Глобальные синглтоны | DI |

## Чек-лист

- [ ] 3 слоя, границы соблюдены.
- [ ] Handler тонкий, зависимости через `Depends`.
- [ ] Сервис не знает про NATS.
- [ ] Репозиторий не коммитит.
- [ ] Subject'ы не захардкожены в сервисах.
