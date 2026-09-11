# FSM и storage (aiogram 3.x)

FSM ведёт пользователя по шагам диалога. Состояние хранится в storage; в проде —
Redis, иначе оно теряется при рестарте.

## Состояния

```python
from aiogram.fsm.state import StatesGroup, State

class Order(StatesGroup):
    waiting_address = State()
    waiting_confirm = State()
```

## Чтение и запись

```python
from aiogram.fsm.context import FSMContext

@router.message(Order.waiting_address)
async def got_address(message, state: FSMContext) -> None:
    await state.update_data(address=message.text)
    await state.set_state(Order.waiting_confirm)

@router.message(Order.waiting_confirm)
async def confirm(message, state: FSMContext) -> None:
    data = await state.get_data()
    await service.create_order(user_id=message.from_user.id, **data)
    await state.clear()
```

- `set_state`/`update_data`/`get_data`/`clear` — основной API.
- Не хранить в FSM большие объекты/секреты — это сериализуется в storage.

## Storage

| Storage | Когда |
|---|---|
| `MemoryStorage` | только dev; теряется при рестарте |
| `RedisStorage` | прод; переживает рестарт/деплой |

```python
from aiogram.fsm.storage.redis import RedisStorage

storage = RedisStorage.from_url(settings.redis_url)
dp = Dispatcher(storage=storage)
```

Redis также удобен для throttling, кэша и фоновых очередей (разные DB-индексы).

## Выход из состояния

Всегда предусматривать выход: команда отмены, таймаут, кнопка «назад», иначе
пользователь «залипает» в шаге.

```python
@router.message(Command("cancel"))
async def cancel(message, state: FSMContext) -> None:
    await state.clear()
    await message.answer("Отменено")
```

## StorageKey

Для чтения state/data в тестах и диагностике используется
`StorageKey(bot_id, chat_id, user_id)`.

## Антипаттерны

| ❌ | ✅ |
|---|---|
| `MemoryStorage` в проде | `RedisStorage` |
| Нет выхода из состояния | отмена/таймаут/назад |
| Секреты/большие объекты в FSM | хранить id/ссылку |
| Логика переходов в хендлерах | сервис + тонкие хендлеры |
| Общий `state` на всех | `StatesGroup` по домену |

## Чек-лист

- [ ] Прод использует `RedisStorage`.
- [ ] Есть выход из каждого состояния.
- [ ] В FSM нет секретов и крупных объектов.
- [ ] Переходы тестируются прогоном апдейтов.
