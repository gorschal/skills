# NATS и FastStream

FastStream — тонкий клиент над `nats-py`. Всё, что умеет NATS, остаётся доступным;
фреймворк берёт на себя lifecycle, сериализацию, ack и документацию.

## Core NATS vs JetStream

| | Core NATS | JetStream |
|---|---|---|
| Персистентность | ❌ | ✅ |
| Ack/redelivery | ❌ | ✅ |
| Масштаб/скорость | максимальная | чуть ниже |
| KV/ObjectStorage | ❌ | ✅ |
| Когда | лёгкий fan-out/RPC | гарантии доставки |

- **Core NATS**: сообщение, опубликованное при отключённом потребителе, теряется;
  подтверждений нет.
- **JetStream**: персистентность, ack/nack, повторная доставка; указывается
  `stream=`.

## Подписка

```python
@broker.subscriber("orders.created", queue="orders-workers")
async def on_order_created(msg: OrderCreated) -> None:
    ...
```

- **Queue group** (`queue=`) — сообщение обрабатывает один потребитель группы;
  масштабирование по горизонтали.
- Без `queue` каждый потребитель получает копию (fan-out).
- Для JetStream — durable consumer/stream.

## Публикация

```python
await broker.publish(OrderCreated(...), subject="orders.created")

# JetStream
await broker.publish(OrderCreated(...), subject="orders.created", stream="orders")
```

## RPC

```python
# blocking request
msg = await broker.request(GetOrderStatus(order_id=order_id), subject="orders.get_status")
status = OrderStatus.model_validate_json(msg.body)
```

Подписчик RPC возвращает значение (или `Response`/`NatsResponse`):

```python
@broker.subscriber("orders.get_status")
async def get_status(query: GetOrderStatus) -> OrderStatus:
    return await service.get_status(query.order_id)
```

- `Response`/`NatsResponse` позволяют добавить headers/`correlation_id`/`stream`.
- `no_reply=True` на subscriber — подавить автоответ.

## Subject'ы

- Subject — просто строка маршрутизации; допустимы wildcard-паттерны в подписке.
- Именование: `<domain>.<entity>.<action>` (`payments.process`, `orders.created`).
- Не хардкодить в сервисах — только handler/config.

## Сообщение и контекст

- `NatsMessage` / FastStream `Context` дают headers, `correlation_id`, `message_id`.
- Трассировку (`correlation_id`, `request_id`) прокидывать в `structlog.contextvars`.

## Гонки и гарантии

- Core NATS не даёт at-least-once — для критичных операций нужен JetStream + `AckPolicy`.
- Повторная доставка возможна → идемпотентность обязательна.
- Один subject с queue group + несколько воркеров = параллельная обработка.

## Антипаттерны

| ❌ | ✅ |
|---|---|
| Критичные операции на Core NATS без ack | JetStream + `AckPolicy` |
| Хардкод subject в сервисе | handler/config |
| Fan-out там, где нужен один обработчик | `queue=` |
| Игнорирование повторной доставки | идемпотентность |
| Subject без схемы именования | `<domain>.<entity>.<action>` |

## Чек-лист

- [ ] Для гарантий — JetStream, а не core NATS.
- [ ] Масштабирование — через queue group.
- [ ] Subject'ы именованы единообразно и не в сервисах.
- [ ] RPC — через `broker.request` с разбором `msg.body`.
- [ ] Трассировочные заголовки прокидываются.
