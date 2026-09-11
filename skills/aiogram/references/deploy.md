# Деплой aiogram-бота

Бот — long-running воркер, а не HTTP-сервис. Основной режим — polling; webhook —
на будущее при высокой нагрузке.

## Polling под systemd

```ini
[Unit]
Description=mybot (aiogram)
After=network-online.target redis-server.service postgresql.service
Wants=network-online.target

[Service]
Type=simple
User=mybot
WorkingDirectory=/opt/mybot
EnvironmentFile=/opt/mybot/.env
ExecStart=/opt/mybot/.venv/bin/python -m bot
Restart=always
RestartSec=3
TimeoutStopSec=30

[Install]
WantedBy=multi-user.target
```

- **Один экземпляр** — вторая копия с тем же токеном → 409.
- `Restart=always` поднимает после краша, но не заменяет обработку ошибок.
- systemd шлёт SIGTERM — бот должен чисто завершиться (`TimeoutStopSec` даёт время).
- Логи — в stdout/stderr → journald (`journalctl -u mybot`).

## RedisStorage для FSM

```python
from aiogram.fsm.storage.redis import RedisStorage

storage = RedisStorage.from_url(settings.redis_url)
dp = Dispatcher(storage=storage)
```

`MemoryStorage` теряет состояние при рестарте.

## Секреты

- `BOT_TOKEN` и прочее — через `EnvironmentFile`/env, читать `config.py`
  (`pydantic-settings`).
- Токен в репозитории/коде — утечка; при компрометации немедленно отозвать.
- `.env` — в `.gitignore`, права `600`, владелец — сервисный пользователь.

## Логирование и наблюдаемость

- Структурные логи в stdout → journald; уровень из env.
- Логировать ошибки Telegram и необработанные исключения.
- Sentry — для алертов по необработанным исключениям.
- Healthcheck для polling: systemd watchdog или внешний «бот жив» (ключ в redis
  с TTL).

## Webhook

```python
from aiohttp import web
from aiogram.webhook.aiohttp_server import SimpleRequestHandler, setup_application

app = web.Application()
SimpleRequestHandler(dispatcher=dp, bot=bot, secret_token=WEBHOOK_SECRET).register(app, path="/webhook")
setup_application(app, dp, bot=bot)
# на старте: await bot.set_webhook(URL, secret_token=WEBHOOK_SECRET,
#                                  drop_pending_updates=True, allowed_updates=...)
```

- **`secret_token` обязателен** — Telegram шлёт его в заголовке
  `X-Telegram-Bot-Api-Secret-Token`; защита от поддельных запросов.
- За nginx с валидным TLS; наружу только HTTPS.
- Один webhook-URL; не держать polling и webhook одновременно.
- `drop_pending_updates` — осознанно.
- Это HTTP-сервис → деплой как веб-приложение, с healthcheck-эндпоинтом.

## Polling vs webhook

|                | Polling             | Webhook                             |
| -------------- | ------------------- | ----------------------------------- |
| Инфраструктура | минимум             | публичный HTTPS, TLS, nginx, secret |
| Масштаб        | один процесс        | можно масштабировать приёмник       |
| Когда          | MVP, небольшие боты | высокая нагрузка                    |

## Антипаттерны

| ❌                         | ✅                                   |
| -------------------------- | ------------------------------------ |
| Вторая polling-копия       | один инстанс                         |
| `MemoryStorage` в проде    | `RedisStorage`                       |
| Токен в коде               | env/`EnvironmentFile`                |
| Webhook без `secret_token` | `secret_token` + TLS                 |
| Polling и webhook вместе   | что-то одно                          |
| Нет graceful shutdown      | `TimeoutStopSec` + закрытие ресурсов |

## Чек-лист

- [ ] Один экземпляр; systemd с `Restart`/`TimeoutStopSec`.
- [ ] FSM на Redis.
- [ ] Секреты из env, `.env` закрыт.
- [ ] Логи структурированы; Sentry/healthcheck настроены.
- [ ] Webhook (если есть) — с `secret_token` за TLS.
