# Ошибки, ack/nack и идемпотентность

FastStream не реализует ретраи и DLQ. Подтверждения — через `AckPolicy`,
повторы — средствами брокера (JetStream redelivery).

## Доменные исключения

```python
class DomainError(Exception): ...

class PaymentAlreadyProcessedError(DomainError):
    def __init__(self, payment_id: UUID) -> None:
        super().__init__(f"Payment {payment_id} already processed")

class InsufficientFundsError(DomainError): ...
```

- В сервисе — только доменные исключения; `HTTPException` и аналоги запрещены.
- `raise ... from e` сохраняет причину.

## AckPolicy

```python
from faststream import AckPolicy

@broker.subscriber("payments.process", ack_policy=AckPolicy.NACK_ON_ERROR)
async def handler(msg: PaymentProcessMessage) -> None: ...
```

| Политика | On success | On error | Применение |
|---|---|---|---|
| `ACK_FIRST` | ack сразу | ack (потеря) | высокий throughput, потеря допустима |
| `ACK` | ack | ack (без повтора) | идемпотентно и не критично |
| `REJECT_ON_ERROR` | ack | reject (без повтора) | ядовитые сообщения |
| `NACK_ON_ERROR` | ack | nack → redelivery | повторяемые операции |
| `MANUAL` | ручной | ручной | полный контроль |

- Разрешение: subscriber > broker > дефолт брокера (NATS: `REJECT_ON_ERROR`).
- Дефолт можно задать на брокере: `NatsBroker(ack_policy=AckPolicy.NACK_ON_ERROR)`.

## Ручное подтверждение

```python
@broker.subscriber("events", ack_policy=AckPolicy.MANUAL)
async def handle_event(msg: EventMessage) -> None:
    try:
        await service.handle(msg)
    except RecoverableError:
        await msg.nack()      # повтор
    except PoisonError:
        await msg.reject()    # в DLQ/отбросить
    else:
        await msg.ack()
```

## Retry и DLQ

- **Retry** — `NACK_ON_ERROR` + JetStream redelivery; backoff/лимиты — настройки
  брокера/стрима, не код сервиса.
- **DLQ** — JetStream/стрим-настройки; ядовитые сообщения → `reject`.
- Не писать собственные циклы повторов в сервисе.

## Идемпотентность

Обязательна для критичных путей: повторная доставка не должна создавать дубль.

```python
async def process(self, message: PaymentProcessMessage) -> None:
    if await self.repo.is_processed(message.payment_id):
        raise PaymentAlreadyProcessedError(message.payment_id)
    async with self.session.begin():
        await self.repo.mark_processed(message.payment_id)
        await self.repo.apply(message)
```

- Уникальный ключ операции (`payment_id`) + проверка «уже обработано».
- Идемпотентная запись (upsert / unique constraint).

## Маппинг ошибок

| Ситуация | Действие |
|---|---|
| Восстановимая (сеть, таймаут) | `NACK_ON_ERROR` → redelivery |
| Доменная, повтор бессмыслен | `REJECT_ON_ERROR`/`reject` (DLQ) |
| Ядовитое сообщение | `reject` + лог |
| Успех | ack |

## Антипаттерны

| ❌ | ✅ |
|---|---|
| Свой retry-цикл в сервисе | `NACK_ON_ERROR`/JetStream |
| Дефолтная ack «на удачу» | Явный `AckPolicy` |
| Нет идемпотентности | Уникальный ключ + проверка |
| `HTTPException` в сервисе | Доменное исключение |
| `reject` для восстановимых ошибок | `nack` |
| Ошибка молча проглатывается | ack/nack/reject + лог |

## Чек-лист

- [ ] `AckPolicy` задан явно и осознанно.
- [ ] Восстановимые ошибки → `NACK_ON_ERROR`.
- [ ] Ядовитые сообщения → reject/DLQ.
- [ ] Критичные обработчики идемпотентны.
- [ ] Нет самодельных retry-циклов.
- [ ] Ошибки логируются с контекстом.
