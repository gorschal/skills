# Асинхронность

async/await для I/O-bound операций. Блокирующий код в async-контексте недопустим.

## Запрещено → замена

| ❌ Запрещено          | ✅ Замена                            |
| --------------------- | ------------------------------------ |
| `requests.get()`      | `httpx.AsyncClient`                  |
| `time.sleep()`        | `await asyncio.sleep()`              |
| `psycopg2`, `sqlite3` | `asyncpg` + SQLAlchemy async         |
| `open()`              | `aiofiles.open()`                    |
| синхронные SDK        | `run_in_threadpool` или async-версия |
| `print()`             | `logger.info(...)`                   |

## async def vs def

- `async def` — если есть хотя бы один `await`.
- `def` — если `await` нет: фреймворк сам выполнит в threadpool.
- Смешивать нельзя: не вызывать блокирующий код внутри `async def` без обёртки.

## Блокирующий код в async

```python
from fastapi.concurrency import run_in_threadpool

async def handler() -> bytes:
    return await run_in_threadpool(blocking_lib.compute, arg)
```

## Конкурентность

```python
import asyncio
import httpx

# Ограниченная конкурентность через TaskGroup (3.11+)
async def fetch_all(urls: list[str]) -> list[str]:
    results: list[str] = []
    async with asyncio.TaskGroup() as tg:
        tasks = [tg.create_task(fetch_one(u)) for u in urls]
    for t in tasks:
        results.append(t.result())
    return results

# Ошибки без падения группы
results = await asyncio.gather(*tasks, return_exceptions=True)
```

- `TaskGroup` — структурная конкурентность: все задачи завершаются/отменяются.
- `gather(..., return_exceptions=True)` — когда нужны все результаты несмотря на ошибки.

## Таймауты

```python
async def fetch(url: str) -> dict:
    async with asyncio.timeout(5):
        async with httpx.AsyncClient(timeout=5.0) as client:
            r = await client.get(url)
            r.raise_for_status()
            return r.json()
```

- Всегда задавайте таймаут на внешние вызовы (сеть, БД, HTTP).
- `asyncio.timeout` (3.11+) предпочтительнее `wait_for`.

## Async context managers и генераторы

```python
from contextlib import asynccontextmanager
from collections.abc import AsyncIterator

@asynccontextmanager
async def get_session() -> AsyncIterator[Session]:
    session = await create_session()
    try:
        yield session
    finally:
        await session.close()

async def read_lines(path: str) -> AsyncIterator[str]:
    async with aiofiles.open(path) as f:
        async for line in f:
            yield line.strip()
```

## Синхронизация

```python
import asyncio

lock = asyncio.Lock()
sem = asyncio.Semaphore(10)          # ограничение конкурентности
event = asyncio.Event()              # сигнал готовности
queue: asyncio.Queue[int] = asyncio.Queue(maxsize=100)
```

## Фоновые задачи

- Не создавайте «висящие» задачи через `asyncio.create_task` без отслеживания.
- Держите реестр задач и отменяйте их на shutdown приложения.
- Для отложенной работы предпочитайте очередь/воркер, а не фон в процессе.

```python
class TaskRegistry:
    def __init__(self) -> None:
        self._tasks: set[asyncio.Task[None]] = set()

    def spawn(self, coro) -> None:
        task = asyncio.create_task(coro)
        self._tasks.add(task)
        task.add_done_callback(self._tasks.discard)

    async def shutdown(self) -> None:
        for task in self._tasks:
            task.cancel()
        await asyncio.gather(*self._tasks, return_exceptions=True)
```

## Антипаттерны

| ❌                                              | ✅                           |
| ----------------------------------------------- | ---------------------------- |
| `time.sleep`, `requests`, `open` в async        | async-аналоги                |
| Блокирующий парсинг большого файла в event loop | `run_in_threadpool`          |
| `create_task` без хранения ссылки               | реестр задач                 |
| Отсутствие таймаутов                            | `asyncio.timeout`            |
| `asyncio.get_event_loop()` в корутине           | `asyncio.get_running_loop()` |
| Смешение sync и async БД                        | один async-драйвер           |

## Чек-лист

- [ ] Нет блокирующих вызовов в `async def`.
- [ ] На все внешние вызовы заданы таймауты.
- [ ] Конкурентные операции — через `TaskGroup`/`gather`.
- [ ] Фоновые задачи отслеживаются и отменяются.
- [ ] `async def` только там, где есть `await`.
