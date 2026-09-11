# Надёжность Telegram API (aiogram 3.x)

Причины, по которым бот «падает / тормозит / дублирует / молчит» в проде.

## Single instance

`getUpdates` может звать **только один** процесс. Две копии → `TelegramConflictError`
(HTTP 409), апдейты теряются.

- Не масштабировать polling горизонтально: один юнит, один процесс.
- Не запускать бота локально при работающем проде с тем же токеном.
- Webhook не конфликтует, но URL один. Не держать polling и webhook одновременно.

## Не блокировать event loop

aiogram полностью async. Блокирующая операция в хендлере останавливает обработку
**всех** апдейтов.

- Только async-библиотеки: `asyncpg`, `httpx`, `redis.asyncio`.
- CPU-тяжёлое — в `run_in_executor`/процесс-пул; долгие задачи — в очередь.
- Признаки: `import requests`, sync-ORM в `async def`, `time.sleep`.

## Ошибки Telegram

aiogram **не** делает авто-retry.

| Исключение               | Когда                              | Что делать                            |
| ------------------------ | ---------------------------------- | ------------------------------------- |
| `TelegramRetryAfter`     | 429, есть `.retry_after`           | подождать `retry_after`, снизить темп |
| `TelegramForbiddenError` | бот заблокирован/кикнут            | пометить неактивным, **не** падать    |
| `TelegramBadRequest`     | битый markup, message not modified | логировать, чинить; часто не ретраить |
| `TelegramNetworkError`   | сеть/таймаут                       | ретрай с backoff                      |
| `TelegramAPIError`       | базовый                            | глобальный error handler              |

Глобальный обработчик обязателен:

```python
@dp.error()
async def on_error(event: ErrorEvent, bot: Bot) -> None:
    logger.exception("handler_error", update_id=event.update.update_id)
```

## Рассылки и троттлинг

Лимиты Telegram (ориентир): ~30 msg/s суммарно, ~1 msg/s в один чат,
~20 msg/min в группу. Наивная рассылка → 429 и риск бана токена.

- Ограничивать темп (≤~25–30/с), паузы; на `TelegramRetryAfter` — sleep.
- Ловить `TelegramForbiddenError`/`TelegramBadRequest` на каждого получателя.
- Большие рассылки — в фоновую задачу/очередь, с паузой/возобновлением и
  учётом уже отправленных (идемпотентность).

## Таймауты

- Разумный таймаут сессии API.
- Long polling: таймаут `getUpdates` штатный, не ставить 0.
- Внешние вызовы — всегда с таймаутом.

## Graceful shutdown

- `start_polling`/`run_polling` ловят SIGINT/SIGTERM.
- Свои ресурсы закрывать в `dp.shutdown`: `bot.session.close()`, redis, пулы БД,
  фоновые задачи (`cancel()` + дождаться).
- Иначе — утечки соединений и потеря незавершённой работы при деплое.

## Идемпотентность

Если бот упал/рестартнул до подтверждения offset, Telegram **повторно** доставит
апдейты. Побочные эффекты (платежи, начисления, внешние вызовы) должны
выдерживать повтор: дедуп по `update_id`/бизнес-ключу, проверка «уже сделано».

## allowed_updates

По умолчанию отдаются не все типы. Если нужны `callback_query`, `my_chat_member`,
`chat_member`, inline — указать явно (или `dp.resolve_used_update_types()`).

## Антипаттерны

| ❌                              | ✅                         |
| ------------------------------- | -------------------------- |
| Два polling-процесса            | один инстанс               |
| Sync-БД/`requests`/`time.sleep` | async                      |
| Рассылка без throttle           | throttle + per-user ошибки |
| `except: pass`                  | глобальный handler + лог   |
| Нет graceful shutdown           | закрывать ресурсы          |
| Игнор повторной доставки        | идемпотентность            |
| Забыт `allowed_updates`         | указать типы               |

## Чек-лист

- [ ] Гарантирован один polling-инстанс.
- [ ] Нет блокирующих вызовов.
- [ ] Глобальный error handler; ошибки Telegram обработаны.
- [ ] Рассылки троттлятся и переживают блокировки.
- [ ] Таймауты заданы.
- [ ] Graceful shutdown закрывает ресурсы.
- [ ] Идемпотентность на побочных эффектах.
- [ ] `allowed_updates` задан.
