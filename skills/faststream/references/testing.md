# Тестирование FastStream

Политика монорепозитория: **BDD — основное сквозное покрытие**; unit — логика
вне BDD и критичные ветки. `TestNatsBroker` — для точечных проверок handler'ов
без реального NATS.

## Границы

| Уровень | Что проверяет |
|---|---|
| BDD | сквозной сценарий: публикация сообщения → эффект |
| Unit (pytest) | бизнес-логика сервисов вне BDD, идемпотентность |
| `TestNatsBroker` | handler без реального брокера (in-memory) |

## Что тестируем unit-тестами

- Бизнес-логику сервисов (ветвления, расчёты, границы).
- Идемпотентность (повторная доставка).
- Расчёты и принятие решений.
- Вызовы зависимостей (через моки).

## Что НЕ тестируем

- То, что покрыто BDD.
- Тонкие handler'ы (обёртки).
- Работу FastStream/NATS и Pydantic-валидацию.
- Простой проброс данных.

## Unit с AsyncMock

```python
from unittest.mock import AsyncMock
import pytest

async def test_process_payment_idempotent() -> None:
    repo = AsyncMock()
    repo.is_processed.return_value = True
    service = PaymentService(session=AsyncMock(), repo=repo)

    with pytest.raises(PaymentAlreadyProcessedError):
        await service.process(PaymentProcessMessage(...))

    repo.apply.assert_not_awaited()
```

## TestNatsBroker (in-memory)

```python
import pytest
from faststream.nats import TestNatsBroker

@pytest.mark.asyncio
async def test_handler_processes_message() -> None:
    async with TestNatsBroker(broker) as br:
        await br.publish(
            {"payment_id": "00000000-0000-0000-0000-000000000001", "amount": 100, "currency": "USD"},
            subject="payments.process",
        )
```

- Не требует реального NATS; работает быстро в CI.
- Полезен для проверки сериализации/handler-wiring.
- Не заменяет BDD для сквозной бизнес-логики.

## Проверка ошибок и ack

- Невалидное сообщение → ожидаемое поведение (reject/логирование).
- Идемпотентность: повторная публикация не создаёт дубль.
- Retry-политика проверяется на уровне брокера/интеграционно (QA).

## Антипаттерны

| ❌ | ✅ |
|---|---|
| Дублировать BDD-сценарий unit-тестом | unit только для пробелов BDD |
| Тестировать тонкий handler | BDD/`TestNatsBroker` |
| Реальный NATS в unit | `AsyncMock`/`TestNatsBroker` |
| Проверка Pydantic-валидации | не тестировать |
| `assert mock.called` | `assert_awaited_once_with(...)` |

## Чек-лист

- [ ] Unit-тесты не дублируют BDD.
- [ ] Покрыта бизнес-логика вне BDD и идемпотентность.
- [ ] Зависимости — `AsyncMock`.
- [ ] `TestNatsBroker` — только для точечных проверок handler'ов.
- [ ] Нет реального NATS в unit.
