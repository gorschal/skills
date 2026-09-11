# Архитектура aiogram-бота (3.x)

Чистая структура снижает баги и упрощает тесты. Роутеры, декларативные фильтры,
middlewares, DI, тонкие хендлеры.

## Router вместо одного Dispatcher

```python
# handlers/start.py
from aiogram import Router, F
from aiogram.filters import CommandStart

router = Router()

@router.message(CommandStart())
async def start(message) -> None: ...

# __main__.py
dp = Dispatcher(storage=storage)
dp.include_router(start.router)
dp.include_router(checkout.router)
```

- Дробить по доменам (start, menu, checkout, admin).
- Порядок включения = порядок проверки; более специфичные фильтры — выше.
- ❌ Всё в одном файле на `@dp.message(...)`.

## Фильтры

```python
from aiogram import F
from aiogram.filters import Command, StateFilter

@router.message(Command("help"))
@router.callback_query(F.data == "open_menu")
@router.message(F.text.regexp(r"^\d+$"))
@router.message(StateFilter(None))            # только вне FSM
```

- Не разбирать `message.text` вручную через `if/elif` — фильтры декларативны и
  тестируемы.
- `F` — «магический» фильтр по атрибутам.

## Middlewares: outer vs inner

```python
router.message.middleware(DBMiddleware())         # inner: после фильтров
router.message.outer_middleware(ThrottleMiddleware())  # outer: на каждый апдейт
```

- **outer** — до роутинга: throttling, бан-чек, логирование.
- **inner** — контекст конкретного хендлера: загрузка пользователя, i18n.
- Не делать тяжёлый блокирующий I/O в middleware — он на каждом апдейте.

## Dependency Injection

```python
# через run_polling — попадёт в kwargs хендлеров
await dp.start_polling(bot, db=db_pool, settings=settings)

# или через middleware
class DBMiddleware(BaseMiddleware):
    async def __call__(self, handler, event, data):
        data["user"] = await get_user(data["db"], event.from_user.id)
        return await handler(event, data)

@router.message()
async def handler(message, db, user) -> None: ...
```

- Не использовать глобальные синглтоны и модульные переменные.
- Зависимости прокидывать явно (`workflow_data`/middleware).

## FSM

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

- Storage конфигурируется на `Dispatcher(storage=...)`.
- Не хранить большие объекты/секреты в FSM-данных.
- Всегда предусматривать выход из состояния (отмена/таймаут).

## Bot и DefaultBotProperties (3.7+)

```python
from aiogram.client.default import DefaultBotProperties
from aiogram.enums import ParseMode

bot = Bot(token, default=DefaultBotProperties(parse_mode=ParseMode.HTML))
```

`Bot(token, parse_mode=...)` — deprecated с 3.7.

## Тонкие хендлеры

Хендлер: распарсил апдейт → вызвал сервис → ответил. Бизнес-логику — в сервисы,
не завязанные на `Message`/`Bot` (тогда её легко тестировать).

## Структура проекта

```
bot/
├── __main__.py        # Bot/Dispatcher, include_router, startup/shutdown, polling
├── config.py          # pydantic-settings; токен НЕ в коде
├── handlers/          # роутеры по доменам
├── keyboards/         # сборка клавиатур
├── middlewares/       # throttling, DB, i18n
├── services/          # бизнес-логика без aiogram-типов
├── states.py          # StatesGroup
└── db/                # модели, репозитории (async)
```

## Антипаттерны

| ❌                           | ✅                         |
| ---------------------------- | -------------------------- |
| Всё в одном файле            | Роутеры по доменам         |
| Логика в хендлере            | Сервис                     |
| `if/elif` по `text`          | Фильтры                    |
| Глобальные синглтоны         | `workflow_data`/middleware |
| Тяжёлый I/O в middleware     | вынести из hot path        |
| `Bot(token, parse_mode=...)` | `DefaultBotProperties`     |

## Чек-лист

- [ ] Роутеры по доменам, порядок фильтров корректен.
- [ ] Хендлеры тонкие, логика в сервисах.
- [ ] Фильтры декларативны.
- [ ] Зависимости через DI, не глобалы.
- [ ] FSM с выходом из состояния.
- [ ] `DefaultBotProperties` вместо deprecated `parse_mode`.
