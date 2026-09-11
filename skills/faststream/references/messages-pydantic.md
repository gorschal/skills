# Сообщения (Pydantic)

Все входящие и исходящие сообщения — Pydantic-модели. FastStream валидирует и
сериализует их автоматически.

## Командные и событийные схемы

```python
from typing import Literal
from uuid import UUID
from pydantic import BaseModel, ConfigDict, Field

class PaymentProcessMessage(BaseModel):
    payment_id: UUID
    amount: int = Field(gt=0)
    currency: str = Field(min_length=3, max_length=3)
    model_config = ConfigDict(extra="forbid")     # команда — строго

class PaymentProcessedEvent(BaseModel):
    payment_id: UUID
    status: Literal["success", "failed"]
    model_config = ConfigDict(from_attributes=True)
```

- **Команды** (входящие) — `extra="forbid"`.
- **События** (исходящие) — `from_attributes=True` при маппинге из домена.
- Ограничения — через `Field`/`Annotated`.

## RPC-ответы

```python
msg = await broker.request(GetOrderStatus(order_id=order_id), subject="orders.get_status")
status = OrderStatus.model_validate_json(msg.body)
```

`broker.request` возвращает `NatsMessage`; тело парсить явно.

## Версионирование

- При эволюции схемы — добавлять поля опционально или вводить новую версию
  subject/модели; не ломать существующих потребителей.
- Новые обязательные поля — только через новый тип события.

## Конверт vs payload

- Простой payload: сама модель как тело сообщения.
- Конверт (метаданные + payload) — если нужны `event_id`, `occurred_at`,
  `version`; тогда оборачивать:

```python
class EventEnvelope[T](BaseModel):     # generic-конверт
    event_id: UUID
    occurred_at: datetime
    payload: T
```

## Валидация

- Валидация — на входе (Pydantic); в сервисе не перепроверять типы.
- Невалидное сообщение не должно «зависать»: определить поведение
  (`REJECT_ON_ERROR`) и логировать.

## Антипаттерны

| ❌                                    | ✅                                |
| ------------------------------------- | --------------------------------- |
| `dict`/`Any` вместо модели            | Pydantic-модель                   |
| Нет `extra="forbid"` на команде       | Явный `extra="forbid"`            |
| Изменение обязательного поля на месте | Новая версия/опциональное поле    |
| Разбор `msg.body` вручную в сервисе   | Разбор в handler, модель в сервис |
| Секреты/PII в событиях                | Минимизировать payload            |

## Чек-лист

- [ ] Все сообщения — Pydantic-модели.
- [ ] Команды `extra="forbid"`; события `from_attributes`.
- [ ] RPC-ответы разбираются через `model_validate_json`.
- [ ] Эволюция схемы без поломки потребителей.
- [ ] PII/секреты в событиях минимизированы.
