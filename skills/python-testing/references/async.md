# Async-тесты

## Настройка

```toml
[tool.pytest.ini_options]
asyncio_mode = "auto"   # async-тесты без явных маркеров
```

Либо явно: `@pytest.mark.asyncio` на каждом async-тесте.

## Async-тест

```python
from unittest.mock import AsyncMock
import pytest

async def test_process_payment_idempotent() -> None:
    repo = AsyncMock()
    repo.is_processed.return_value = True
    service = PaymentService(repo=repo)

    with pytest.raises(PaymentAlreadyProcessedError):
        await service.process(Payment(id=1))

    repo.apply.assert_not_awaited()
```

- Зависимости — `AsyncMock`; проверки — `assert_awaited_once_with`.
- `pytest.raises` вокруг `await`.

## Async-фикстуры

```python
import pytest
from collections.abc import AsyncIterator

@pytest.fixture
async def repo() -> AsyncIterator[AsyncMock]:
    r = AsyncMock()
    yield r
    # cleanup
```

- Очистка — `yield` в async-фикстуре.
- `scope="function"` по умолчанию.

## Конкурентность

```python
import asyncio

async def test_concurrent_fetch(monkeypatch) -> None:
    results = await asyncio.gather(fetch(1), fetch(2))
    assert len(results) == 2
```

- Не полагаться на реальные таймеры; при необходимости патчить/инъектировать.
- Избегать флаки из-за гонок: детерминированные моки, `asyncio.Event` при нужде.

## Правила

- Async-код тестируется в async-тестах, не через `asyncio.run` внутри sync-теста.
- Не блокировать event loop в тестах (никаких `time.sleep`).
- Не тестировать сам event loop — тестировать логику.

## Антипаттерны

| ❌                         | ✅                             |
| -------------------------- | ------------------------------ |
| `MagicMock` для async      | `AsyncMock`                    |
| `time.sleep` в async-тесте | фейки/детерминизм              |
| Реальные таймеры/гонки     | моки/события                   |
| `asyncio.run` в sync-тесте | async-тест/`asyncio_mode=auto` |

## Чек-лист

- [ ] `pytest-asyncio` настроен.
- [ ] Async-зависимости — `AsyncMock`.
- [ ] Очистка через async `yield`.
- [ ] Тесты детерминированы, без реальных таймеров.
