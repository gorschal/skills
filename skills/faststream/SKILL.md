---
name: faststream
description: >
  Use when building event-driven services with FastStream 0.7.x and NATS:
  subscribers/publishers, Pydantic messages, dependency injection, AckPolicy
  (ack/nack/reject), JetStream, RPC, idempotency, structlog, in-memory tests.
  Триггеры: FastStream, NATS, JetStream, broker, subscriber, publisher,
  ack_policy, queue group, AsyncAPI, TestNatsBroker, event-driven.
  Для HTTP — навык fastapi; общие практики Python — python.
license: MIT
compatibility: opencode
metadata:
  version: "1.0.0"
  domain: backend
  triggers: FastStream, NATS, JetStream, broker, subscriber, publisher, ack_policy, queue group, AsyncAPI, event-driven
  role: specialist
  scope: implementation
  output-format: code
  related-skills: python, python-testing, pytest-bdd, fastapi, security, postgres
---

# FastStream

FastStream 0.7.x для event-driven сервисов на NATS: тонкие handler'ы,
сервисный слой, Pydantic-сообщения, явный контроль ack/nack и идемпотентность.

> FastStream — **тонкий клиент** над брокером: он не реализует ретраи,
> отложенную доставку и оркестрацию. Это архитектурные решения на уровне
> приложения/брокера. Общие практики языка — навык `python`.

## Когда применять

- Подписки/публикации, NATS (core/JetStream), queue groups.
- Pydantic-сообщения, DI, AckPolicy, идемпотентность.
- RPC, AsyncAPI-документация, тесты без брокера.

## Ключевые принципы

1. **3 слоя:** Handler → Service → Repository. Границы неприкосновенны.
2. **Handler тонкий**: принимает сообщение, вызывает сервис через `Depends`.
3. **Бизнес-логика и идемпотентность — в сервисе.**
4. **Репозиторий — только доступ к данным** (SQLAlchemy Core/ORM), без `commit`.
5. **Сообщения — Pydantic-модели**; `extra="forbid"` на командных схемах.
6. **Ack-политика задаётся явно** (`AckPolicy`), а не «по умолчанию».
7. **Критичные обработчики идемпотентны** (уникальный ключ/проверка).
8. **DI через `Depends`**; ручное создание сервисов запрещено.
9. **Тесты:** BDD — основное покрытие; unit — логика вне BDD; `TestNatsBroker` —
  для точечных проверок handler'ов.

## Архитектура (3 слоя)

| Слой | Файл | Разрешено |
|---|---|---|
| Handler | `*handler.py` / `*router.py` | подписка, `Depends()`, один вызов сервиса |
| Service | `*service.py` | бизнес-логика, идемпотентность, транзакции, доменные исключения |
| Repository | `*repository.py` | только доступ к данным, без `commit`/`rollback` |
| Schemas | `*schemas.py` | Pydantic-модели сообщений |

```python
@broker.subscriber("payments.process", queue="payments-workers")
async def process_payment_handler(
    message: PaymentProcessMessage,
    service: PaymentService = Depends(get_payment_service),
) -> None:
    await service.process(message)
```

- Handler не содержит бизнес-логики и прямых обращений к БД/брокеру.
- Сервис не знает про NATS (subject, headers); принимает данные.
- Репозиторий не коммитит; транзакция — в сервисе.
- Схема/миграции — Django; сервис не создаёт таблицы (`create_all`/свои миграции запрещены).

Подробно: [references/architecture.md](references/architecture.md).

## Dependency Injection

```python
async def get_db() -> AsyncGenerator[AsyncSession, None]:
    async with async_session_maker() as session:
        yield session

def get_payment_service(repo: PaymentRepository = Depends(get_payment_repository)) -> PaymentService:
    return PaymentService(repo)
```

- ❌ `service = PaymentService(db)` внутри handler.
- FastStream также даёт `Logger` и `Context` через `Depends`/аннотации.

## NATS и FastStream

- **Core NATS**: без персистентности и подтверждений — сообщение теряется, если
  потребитель отключён.
- **JetStream**: персистентность, ack, redelivery, KeyValue/ObjectStorage.
- **Queue groups** (`queue="..."`) — горизонтальное масштабирование: сообщение
  обрабатывает один потребитель группы.

```python
await broker.publish(OrderCreated(...), subject="orders.created")

@broker.subscriber("orders.created", queue="orders-workers")
async def on_order_created(msg: OrderCreated) -> None: ...

# RPC
msg = await broker.request(GetOrderStatus(order_id=...), subject="orders.get_status")
status = OrderStatus.model_validate_json(msg.body)
```

- Subject'ы не хардкодить в сервисах — только в handler/config.
- Для JetStream указывать `stream=`.

Подробно: [references/nats.md](references/nats.md).

## Сообщения (Pydantic)

```python
class PaymentProcessMessage(BaseModel):
    payment_id: UUID
    amount: int = Field(gt=0)
    model_config = ConfigDict(extra="forbid")
```

- Все входящие/исходящие сообщения — Pydantic-модели.
- `extra="forbid"` на командных схемах; версионирование сообщений при эволюции.
- Десериализация RPC-ответов — через `model_validate_json(msg.body)`.

Подробно: [references/messages-pydantic.md](references/messages-pydantic.md).

## Ack/Nack и надёжность

FastStream управляет подтверждениями через `AckPolicy`:

```python
@broker.subscriber("payments.process", ack_policy=AckPolicy.NACK_ON_ERROR)
async def handler(msg: PaymentProcessMessage) -> None: ...
```

| Политика | On error |
|---|---|
| `ACK_FIRST` | подтверждает сразу (риск потери) |
| `ACK` | подтверждает после обработки, даже при ошибке |
| `REJECT_ON_ERROR` | отклоняет сообщение (без повтора) |
| `NACK_ON_ERROR` | nack → redelivery (повтор) |
| `MANUAL` | ручной `msg.ack()`/`msg.nack()`/`msg.reject()` |

- Разрешение: subscriber > broker > дефолт брокера (NATS: `REJECT_ON_ERROR`).
- **Retry/DLQ — не в FastStream**, а через `NACK_ON_ERROR`/JetStream redelivery
  и настройки брокера. В сервисе retry-логику не писать.
- Идемпотентность обязательна: повторная доставка не должна создавать дубль.

Подробно: [references/errors-ack.md](references/errors-ack.md).

## Обработка ошибок

```python
class DomainError(Exception): ...
class PaymentAlreadyProcessedError(DomainError): ...
class InsufficientFundsError(DomainError): ...
```

- В сервисе — только доменные исключения; `HTTPException` и аналоги запрещены.
- Транспортный маппинг (ack/nack/reject) — через `AckPolicy`/middleware.
- `raise ... from e` сохраняет причину.

## Конфигурация и логирование

- `pydantic-settings`; `os.getenv()` вне config запрещён; секреты — из окружения.
- structlog: события `snake_case`, параметры `key=value`, без `print`/f-строк.
- Контекст (`request_id`, `correlation_id`) — через `structlog.contextvars` /
  FastStream `Context`; очищать в `finally`.

## Тестирование

- **BDD — основное покрытие** (публикация сообщения → проверка эффекта).
- Unit — бизнес-логика сервисов вне BDD и критичные ветки; зависимости `AsyncMock`.
- `TestNatsBroker` — in-memory проверка handler'ов без реального NATS.
- Не тестировать Pydantic-валидацию и тонкий проброс данных.

Подробно: [references/testing.md](references/testing.md).

## Инструменты

```bash
uv run faststream run app.main:app --reload
uv run faststream run app.main:app --workers 3
uv run ruff check . && uv run ruff format --check .
uv run pyright
uv run pytest -v
```

## Запрещённые паттерны

| ❌ Запрещено | ✅ Правильно |
|---|---|
| Бизнес-логика в handler | Вызов service |
| SQL/доступ к БД в handler | Repository |
| Ручное создание сервиса | `Depends(get_..._service)` |
| `commit()` в repository | Транзакция в сервисе |
| Хардкод subject'ов в service | Subject только в handler/config |
| Отсутствие идемпотентности | Проверка + уникальный ключ |
| Дефолтная ack-политика «на удачу» | Явный `AckPolicy` |
| Retry-логика в сервисе | `NACK_ON_ERROR`/JetStream redelivery |
| `print` / f-строки в логах | `logger.info("event", key=value)` |
| Синхронные драйверы/`requests` | async-аналоги |

## Чек-лист code review

- [ ] Handler только вызывает сервис через `Depends`.
- [ ] Бизнес-логика и идемпотентность — в сервисе.
- [ ] Repository использует SQLAlchemy и не делает `commit`.
- [ ] Транзакции — в сервисе (`async with session.begin()`).
- [ ] Все сообщения — Pydantic-модели; `extra="forbid"` где нужно.
- [ ] `AckPolicy` задан явно; `NACK_ON_ERROR` для повторяемых операций.
- [ ] Доменные исключения вместо кодов возврата.
- [ ] Логи через structlog, события `snake_case` + контекст.
- [ ] Нет блокирующих вызовов.
- [ ] Полная типизация, проходит `pyright`.
- [ ] Unit-тесты не дублируют BDD; зависимости замоканы.
- [ ] `ruff check`, `ruff format --check`, `pyright`, тесты проходят.

## Справочники

| Тема | Reference | Загружать когда |
|---|---|---|
| Архитектура, слои, DI | [references/architecture.md](references/architecture.md) | Проектирование handler'ов/сервисов |
| NATS и FastStream | [references/nats.md](references/nats.md) | Subjects, queue groups, JetStream, RPC |
| Сообщения (Pydantic) | [references/messages-pydantic.md](references/messages-pydantic.md) | Схемы сообщений, версионирование |
| Ошибки, ack/nack, идемпотентность | [references/errors-ack.md](references/errors-ack.md) | AckPolicy, redelivery, DLQ, повторы |
| Жизненный цикл, контекст, middleware | [references/lifecycle-context.md](references/lifecycle-context.md) | lifespan, Context, Logger, observability |
| Тестирование | [references/testing.md](references/testing.md) | Unit, BDD, TestNatsBroker |

## Связанные навыки

- `python` — общие практики языка, async, ошибки, логи, инструменты.
- `python-testing` / `pytest-bdd` — тестовая инфраструктура и BDD.
- `fastapi` — HTTP-часть того же сервиса.
- `postgres` — БД под сервисом.
