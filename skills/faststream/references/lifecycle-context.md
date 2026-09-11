# Жизненный цикл, контекст, middleware

## Приложение и брокер

```python
from faststream import FastStream
from faststream.nats import NatsBroker

broker = NatsBroker(settings.nats_url)
app = FastStream(broker)
```

Запуск:

```bash
faststream run app.main:app --reload        # dev
faststream run app.main:app --workers 3     # масштабирование
```

`FastStream` — цель CLI. При встраивании в другое приложение (FastAPI) запускать
брокер в его lifespan: `await broker.start()` / `await broker.stop()`.

## Lifespan

```python
from contextlib import asynccontextmanager
from faststream import FastStream

@asynccontextmanager
async def lifespan(context) -> AsyncGenerator[None, None]:
    # startup: пул БД, HTTP-клиент
    yield
    # shutdown: закрыть ресурсы
```

- Ресурсы создаются на старте и закрываются на остановке.
- Graceful shutdown: корректно завершить обработку, закрыть соединения.

## Context

FastStream `Context` даёт доступ к метаданным сообщения (headers, `correlation_id`,
`message_id`):

```python
from faststream import Context
from faststream.nats import NatsMessage

@broker.subscriber("orders.created")
async def handler(
    msg: OrderCreated,
    raw: NatsMessage = Context(),
) -> None:
    logger.info("order_created", correlation_id=raw.correlation_id)
```

Прокидывать `correlation_id`/`request_id` в `structlog.contextvars` для трассировки.

## Logger

FastStream предоставляет `Logger` через DI:

```python
from faststream import Logger

@broker.subscriber("orders.created")
async def handler(msg: OrderCreated, logger: Logger) -> None:
    logger.info("order_created", order_id=str(msg.order_id))
```

В проекте — единый structlog; `Logger` использовать как обёртку над ним.

## Middleware

- Middleware — для сквозных задач: логирование, метрики, трассировка, error
  handling, ручной ack.
- Не помещать бизнес-логику в middleware.
- Исключения маппить централизованно (например, `REJECT_ON_ERROR` для
  неразбираемых сообщений).

## Observability и AsyncAPI

- OpenTelemetry/Prometheus — через штатные middleware FastStream.
- Health-проверки — отдельный endpoint/проба.
- AsyncAPI-документация генерируется автоматически — держать её актуальной.

## Config management

- Настройки — `pydantic-settings` из env; `os.getenv()` вне config запрещён.
- Subject'ы, stream'ы, лимиты — в конфиге, не в сервисах.

## Антипаттерны

| ❌ | ✅ |
|---|---|
| Ресурсы-глобалы без закрытия | lifespan + shutdown |
| Бизнес-логика в middleware | только сквозные задачи |
| `print`/f-строки | `logger.info("event", key=value)` |
| `correlation_id` игнорируется | прокидывать в contextvars |
| Subject'ы в сервисах | config/handler |
| `os.getenv()` вне config | `pydantic-settings` |

## Чек-лист

- [ ] Ресурсы создаются/закрываются в lifespan.
- [ ] Graceful shutdown реализован.
- [ ] `Context`/`correlation_id` прокидываются в логи.
- [ ] Middleware — только сквозные задачи.
- [ ] AsyncAPI/observability включены.
- [ ] Subject'ы и настройки — в config.
