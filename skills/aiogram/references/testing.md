# Тестирование aiogram-бота (3.x)

Главный принцип: **бизнес-логику тестируем отдельно от Telegram**, хендлеры —
точечно через `dp.feed_update` с моком `Bot`.

## Границы

| Уровень | Что проверяет |
|---|---|
| BDD / unit | бизнес-логика в сервисах (без aiogram) |
| Handler-тесты | реакция хендлера на апдейт (`feed_update`) |
| FSM-тесты | переходы состояний и данные |

## 1. Логику — из хендлеров в сервисы

Если логика в `services/` и не зависит от `Message`/`Bot`, она тестируется
обычным pytest/BDD без моков Telegram:

```python
def test_order_total() -> None:
    assert calculate_total([...]) == 100
```

Это покрывает большую часть рисков.

## 2. Хендлер через мок Bot и feed_update

```python
import pytest
from unittest.mock import AsyncMock
from aiogram import Dispatcher
from aiogram.types import Update, Message, Chat, User
from aiogram.fsm.storage.memory import MemoryStorage

@pytest.fixture
def bot() -> AsyncMock:
    b = AsyncMock()
    b.id = 42
    return b

@pytest.fixture
def dp() -> Dispatcher:
    dp = Dispatcher(storage=MemoryStorage())
    dp.include_router(start.router)
    return dp

async def test_start_replies(dp, bot) -> None:
    msg = Message(
        message_id=1, date=..., chat=Chat(id=1, type="private"),
        from_user=User(id=1, is_bot=False, first_name="A"), text="/start",
    )
    await dp.feed_update(bot, Update(update_id=1, message=msg))
    bot.send_message.assert_awaited()
```

- Проверять **эффект** (`send_message.assert_awaited()`), а не «не упало».
- Сетевые вызовы Telegram замоканы — тесты не ходят в реальный API.

## 3. Переходы FSM

```python
async def test_order_flow(dp, bot) -> None:
    await dp.feed_update(bot, _msg("/order"))
    await dp.feed_update(bot, _msg("ул. Пушкина"))
    # проверить, что состояние очищено, а данные сохранены через сервис
```

- `MemoryStorage` для тестов; при необходимости читать state/data через
  `StorageKey(bot_id, chat_id, user_id)`.
- Проверять и успешный путь, и негативный (неверный ввод, отмена).

## 4. Сторонние помощники

Community-библиотеки (например, `aiogram_tests`) с готовым `MockedBot` удобны, но
проверяйте совместимость с вашей версией aiogram 3.x. Базовый подход
(`AsyncMock` + `feed_update`) работает без зависимостей.

## Что проверять

- Бизнес-логика вынесена и покрыта (не всё в хендлерах).
- Тесты проверяют эффект, а не отсутствие исключения.
- Сетевые вызовы замоканы.
- Негативные сценарии: заблокированный пользователь, неверный ввод в FSM, flood.

## Антипаттерны

| ❌ | ✅ |
|---|---|
| Вся логика в хендлерах (нетестируемо) | Сервисы + тонкие хендлеры |
| Проверять «не упало» | `assert_awaited`/проверка state |
| Реальный Telegram в тестах | `AsyncMock` Bot |
| Тестировать только утилиты | хендлеры и FSM тоже |
| Нет негативных сценариев | блокировка, неверный ввод, flood |

## Чек-лист

- [ ] Бизнес-логика вынесена и покрыта без aiogram.
- [ ] Хендлеры проверяются через `feed_update`.
- [ ] Тесты проверяют эффект, а не только отсутствие ошибки.
- [ ] Сетевые вызовы замоканы.
- [ ] Покрыты переходы FSM и негативные сценарии.
