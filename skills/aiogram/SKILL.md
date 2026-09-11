---
name: aiogram
description: >
  Use when building Telegram bots with aiogram 3.x: Routers, filters, middlewares,
  FSM and storage, thin handlers, flood-control, graceful shutdown, idempotency,
  polling/webhook deploy, and tests. Триггеры: aiogram, Telegram bot, Router,
  Dispatcher, FSM, RedisStorage, callback_query, webhook, polling, flood control,
  TelegramRetryAfter, feed_update. Только aiogram 3.x.
license: MIT
compatibility: opencode
metadata:
  version: "1.0.0"
  domain: backend
  triggers: aiogram, Telegram bot, Router, Dispatcher, FSM, RedisStorage, webhook, polling, flood control
  role: specialist
  scope: implementation
  output-format: code
  related-skills: python, python-testing, pytest-bdd, faststream, fastapi
---

# aiogram

Telegram-боты на aiogram 3.x: тонкие хендлеры, роутеры по доменам, FSM на Redis,
корректная обработка flood-control и graceful shutdown.

> Только aiogram 3.x. Код на 2.x (`executor`, `Dispatcher(bot)`,
> `@dp.message_handler`) — сначала миграция на 3.x.

## Когда применять

- Написание/ревью бота: роутеры, фильтры, middlewares, FSM, клавиатуры.
- Обработка ошибок Telegram, рассылки, throttling.
- Деплой (polling/webhook), тесты хендлеров и FSM.

## Ключевые принципы

1. **Роутеры по доменам**, не один `Dispatcher` с десятками `@dp.message`.
2. **Хендлер тонкий**: распарсил → вызвал сервис → ответил.
3. **Бизнес-логика — в сервисах**, не завязанных на `Message`/`Bot`.
4. **Фильтры декларативны** (`F`, `Command`), не `if/elif` по `message.text`.
5. **Один polling-процесс** (две копии → 409 Conflict).
6. **Не блокировать event loop**: только async-библиотеки.
7. **FSM в проде — Redis**, не `MemoryStorage`.
8. **Graceful shutdown**: закрывать `bot.session`, redis, пулы, задачи.
9. **Идемпотентность** на побочных эффектах (повторная доставка апдейтов).
10. **Тесты:** BDD — бизнес-логика; хендлеры — `feed_update` с моком `Bot`.

## Архитектура

```python
# handlers/start.py
from aiogram import Router, F
from aiogram.filters import CommandStart

router = Router()

@router.message(CommandStart())
async def start(message, user) -> None:
    await message.answer(f"Привет, {user.first_name}")

# __main__.py
dp = Dispatcher(storage=storage)
dp.include_router(start.router)
dp.include_router(checkout.router)
```

| Компонент | Ответственность |
|---|---|
| Routers | хендлеры по домену; порядок включения = порядок проверки |
| Filters | декларативный отбор апдейтов (`Command`, `F`, `StateFilter`) |
| Middlewares | outer — throttling/бан/логи; inner — контекст хендлера (БД, i18n) |
| Services | бизнес-логика без aiogram-типов |
| Keyboards | сборка клавиатур |
| States | `StatesGroup` |

- Более специфичные фильтры — выше.
- Зависимости — через `workflow_data`/middleware (`data["db"]`), не глобалы.
- `Bot` создавать с `DefaultBotProperties` (не deprecated `parse_mode=`).

Подробно: [references/architecture.md](references/architecture.md).

## FSM и storage

```python
from aiogram.fsm.state import StatesGroup, State
from aiogram.fsm.context import FSMContext

class Order(StatesGroup):
    waiting_address = State()

@router.message(Order.waiting_address)
async def got_address(message, state: FSMContext) -> None:
    await state.update_data(address=message.text)
    await state.clear()
```

- Прод — `RedisStorage.from_url(...)`; `MemoryStorage` — только dev.
- Не хранить в FSM большие объекты/секреты.
- Всегда предусматривать выход из состояния (отмена/таймаут).

Подробно: [references/fsm-storage.md](references/fsm-storage.md).

## Надёжность

- **Single instance**: `getUpdates` зовёт только один процесс; иначе
  `TelegramConflictError` (409) и потеря апдейтов. Webhook — тоже один URL.
- **Не блокировать loop**: `requests`, `time.sleep`, sync-драйверы, тяжёлый CPU —
  запрещены в хендлерах. Async-библиотеки; тяжёлое — в очередь/executor.
- **`allowed_updates`**: если нужны `callback_query`, `my_chat_member` и т.п. —
  указать (или `dp.resolve_used_update_types()`).

### Ошибки Telegram (`aiogram.exceptions`)

| Исключение | Действие |
|---|---|
| `TelegramRetryAfter` (429) | подождать `.retry_after`, снизить темп |
| `TelegramForbiddenError` | пометить получателя неактивным, не падать |
| `TelegramBadRequest` | логировать, чинить причину (часто не ретраить) |
| `TelegramNetworkError` | ретрай с backoff |
| `TelegramAPIError` | глобальный error handler |

- **Рассылки**: троттлить (~≤25–30 msg/s, ~1/s в чат), ловить ошибки на каждого
  получателя, выносить в фоновую очередь с идемпотентностью.
- **Таймауты** на все внешние вызовы.
- **Graceful shutdown**: в `dp.shutdown` закрыть `bot.session`, redis, пулы, задачи.

Подробно: [references/reliability.md](references/reliability.md).

## Обработка ошибок

- Глобальный error handler на `Dispatcher` (`@dp.error()`/`dp.errors.register`) —
  иначе исключение роняет обработку апдейта без следа.
- Доменные исключения из сервисов логировать; пользователю — понятное сообщение.
- Не проглатывать ошибки (`except: pass` запрещён).

## Конфигурация и логирование

```python
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    bot_token: SecretStr
    redis_url: str
    model_config = SettingsConfigDict(env_file=".env")

settings = Settings()
```

- `BOT_TOKEN` — только из env; в коде/репозитории — утечка.
- structlog: события `snake_case`, параметры `key=value`, без `print`/f-строк.
- Ошибки Telegram и необработанные исключения — логировать; Sentry для алертов.

## Тестирование

- **Бизнес-логика** — в сервисах, тестируется обычным pytest/BDD без aiogram.
- **Хендлеры** — прогон апдейта через `dp.feed_update` с `AsyncMock` Bot;
  проверять **эффект** (`send_message.assert_awaited()`), а не «не упало».
- **FSM** — последовательность апдейтов с `MemoryStorage`, проверка state/data.
- Негативные сценарии: заблокированный пользователь, неверный ввод, flood.

```python
async def test_start_replies(dp, bot) -> None:
    await dp.feed_update(bot, Update(update_id=1, message=_msg("/start")))
    bot.send_message.assert_awaited()
```

Подробно: [references/testing.md](references/testing.md).

## Деплой

| | Polling | Webhook |
|---|---|---|
| Инфраструктура | минимум | публичный HTTPS, TLS, nginx, secret |
| Масштаб | один процесс | можно масштабировать приёмник |
| Когда | по умолчанию, MVP | высокая нагрузка |

- Polling: systemd-юнит, `Restart=always`, `TimeoutStopSec` для graceful shutdown.
- Webhook: aiohttp + `SimpleRequestHandler(secret_token=...)` за nginx/TLS;
  `secret_token` обязателен; не держать polling и webhook одновременно.

Подробно: [references/deploy.md](references/deploy.md).

## Запрещённые паттерны

| ❌ Запрещено | ✅ Правильно |
|---|---|
| Всё в одном файле на `@dp.message` | Роутеры по доменам |
| Бизнес-логика в хендлере | Сервис |
| `if/elif` по `message.text` | Фильтры (`F`, `Command`) |
| `time.sleep`/`requests`/sync-драйверы | async-аналоги |
| `MemoryStorage` в проде | `RedisStorage` |
| Две polling-копии | один инстанс |
| Рассылка без throttle и обработки ошибок | throttle + per-user обработка |
| `except: pass` | глобальный error handler + лог |
| Токен в коде/репозитории | env/`EnvironmentFile` |
| Забыть `allowed_updates` | явно указать типы |
| Отсутствие идемпотентности | дедуп по `update_id`/бизнес-ключу |
| `Bot(token, parse_mode=...)` | `DefaultBotProperties` |

## Чек-лист code review

- [ ] Роутеры по доменам; порядок фильтров корректен.
- [ ] Хендлеры тонкие; логика в сервисах.
- [ ] Фильтры декларативны, без ручного разбора `text`.
- [ ] Нет блокирующих вызовов в хендлерах.
- [ ] FSM на `RedisStorage` в проде; есть выход из состояния.
- [ ] Один polling-инстанс; `allowed_updates` задан.
- [ ] Глобальный error handler; ошибки Telegram обработаны.
- [ ] Рассылки троттлятся и обрабатывают ошибки на получателя.
- [ ] Graceful shutdown закрывает сессию/redis/пулы.
- [ ] Идемпотентность на побочных эффектах.
- [ ] Токен/секреты — из env.
- [ ] Логи structlog; тесты проверяют эффект.
- [ ] `ruff check`, `ruff format --check`, `pyright`, тесты проходят.

## Справочники

| Тема | Reference | Загружать когда |
|---|---|---|
| Архитектура, роутеры, middlewares, DI | [references/architecture.md](references/architecture.md) | Структура бота, фильтры, зависимости |
| Надёжность, ошибки, flood-control | [references/reliability.md](references/reliability.md) | 409/429, рассылки, shutdown, идемпотентность |
| FSM и storage | [references/fsm-storage.md](references/fsm-storage.md) | Состояния, Redis/Memory, выходы |
| Деплой | [references/deploy.md](references/deploy.md) | systemd, webhook+nginx, секреты, healthcheck |
| Тестирование | [references/testing.md](references/testing.md) | feed_update, FSM, моки Bot |

## Связанные навыки

- `python` — общие практики языка, async, ошибки, логи, инструменты.
- `python-testing` / `pytest-bdd` — тестовая инфраструктура и BDD.
- `faststream` — фоновая очередь для рассылок/задач.
- `fastapi` — если у бота есть HTTP-часть/webhook-приложение.
