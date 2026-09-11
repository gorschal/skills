---
name: python
description: >
  Use when writing or reviewing production Python 3.12+ code: layered
  architecture, full type hints, async/await, domain error handling,
  configuration and secrets, structlog, security, pytest unit tests,
  Google-style docstrings, and tooling (uv, ruff, pyright). Триггеры: Python,
  типизация, Pyright, mypy, asyncio, pytest, structlog, ruff, uv, pydantic,
  dataclass, рефакторинг Python, code review Python, безопасность Python.
  Для Django использовать навык django; для FastAPI — fastapi.
license: MIT
compatibility: opencode
metadata:
  version: "1.2.0"
  domain: language
  triggers: Python, typing, Pyright, mypy, asyncio, pytest, structlog, ruff, uv, pydantic, architecture, security
  role: specialist
  scope: implementation
  output-format: code
  related-skills: python-testing, django, fastapi, security, documentation, python-audit
---

# Python

Базовый навык для production-кода на Python 3.12+: слоистая архитектура,
типизация, async, доменные ошибки, structlog, безопасность, unit-тесты, docstrings,
инструменты (`uv`, `ruff`, `pyright`).

> Язык и общие практики. Фреймворк-специфика — `django`/`fastapi`; тесты —
> `python-testing`/`pytest-bdd`.

## Когда применять

- Написание/рефакторинг Python: сервисы, репозитории, схемы, утилиты.
- Настройка типизации (`pyright`), линтинга (`ruff`), проекта (`uv`).
- Слои, DI, транзакции, обработка ошибок, логирование.
- НЕ для Django/FastAPI-специфики — переходить в `django`/`fastapi`.

## Ключевые принципы

1. **Код без аннотаций типов — брак.** Все аргументы/возвраты типизированы;
   проверка — `pyright` (strict).
2. **BDD — основное покрытие**, unit — логика вне BDD и критичные ветки.
3. **Ошибки — доменными исключениями**, не кодами возврата. `raise low, catch high`.
4. **Границы слоёв неприкосновенны:** presentation не знает SQL; repository без
   бизнес-логики; service без HTTP/`request`.
5. **Транзакции — только в сервисе**; репозиторий не коммитит.
6. **Секреты — из окружения.** Хардкод и `os.getenv()` вне `config` запрещены.
7. **Логи — structlog**, события `snake_case` прошедшего времени, без `print`.
8. **Не изобретать паттерны**; отклонение — объяснять.

## Архитектура (слои)

| Слой | Ответственность | Не имеет права |
|---|---|---|
| Presentation | вход/выход, валидация, один вызов сервиса | SQL, бизнес-логика, `try/except` бизнес-ошибок |
| Service | бизнес-логика, транзакции, оркестрация, доменные исключения | `request`/`response`, прямой SQL |
| Repository | доступ к данным, только запросы | бизнес-логика, `commit`/`rollback` |
| Schema/DTO | форма и валидация данных | побочные эффекты |
| Domain/Config | конфиг, исключения, логи, метаданные | прикладная логика |

- Зависимости — явно через `__init__`; fallback на репозиторий запрещён.
- Репозиторий типизируется `Protocol` → простые моки.
- Транзакция охватывает всю мутацию; побочные эффекты — после коммита
  (`on_commit`/outbox).

Подробно: [references/architecture.md](references/architecture.md).

## Типизация

- `X | None` (не `Optional[X]`), `list[str]` (не `List[str]`).
- `Sequence`/`Mapping` — для чтения; `list`/`dict` — для мутации.
- `TypedDict` — строки БД/payload; `Protocol` — интерфейсы.
- `Literal`, `Final`, `TypeAlias`, `Self`, `@overload` — по назначению.
- `Any` — только на границе с нетипизированной библиотекой.

Подробно: [references/typing.md](references/typing.md).

## Асинхронность

| ❌ Запрещено | ✅ Замена |
|---|---|
| `requests.get()` | `httpx.AsyncClient` |
| `time.sleep()` | `await asyncio.sleep()` |
| `psycopg2`, `sqlite3` | `asyncpg` + SQLAlchemy async |
| `open()` | `aiofiles.open()` |

- `async def` — если есть `await`; иначе `def`.
- Блокирующее в async — только `run_in_threadpool`.
- Группы — `asyncio.TaskGroup`; таймауты — `asyncio.timeout()`.
- Фоновые задачи отслеживаются и отменяются на shutdown.

Подробно: [references/async.md](references/async.md).

## Обработка ошибок

```python
class DomainError(Exception): ...
class UserNotFoundError(DomainError): ...

async def get_user(self, user_id: int) -> User:
    user = await self.repo.get(user_id)
    if user is None:
        raise UserNotFoundError(user_id)
    return user
```

- В сервисе — только доменные исключения; `HTTPException` запрещён.
- Маппинг на транспорт — централизованно (handler/middleware/ack-nack).
- `raise ... from original` — сохранять причину.
- Ловить только чтобы: повторить, трансформировать, очистить, добавить контекст.

Подробно: [references/errors-logging.md](references/errors-logging.md).

## Конфигурация и секреты

```python
class Settings(BaseSettings):
    database_url: str
    secret_key: SecretStr
    model_config = SettingsConfigDict(env_file=".env")
```

- Всё окружение — через `Settings`; `os.getenv()` вне `config.py` запрещён.
- Секреты не логируются, не в ответах, не в git; `.env` в `.gitignore`,
  `.env.example` без значений.

## Логирование

| ✅ | ❌ |
|---|---|
| `logger.info("user_created", user_id=id)` | `print(f"User {id} created")` |
| `logger.exception(...)` в `except` | f-строки в логах |
| события `snake_case` прошедшего времени | «User Created» |

- Контекст (`request_id`, `user_id`) — `structlog.contextvars`; очищать в `finally`.

Подробно: [references/errors-logging.md](references/errors-logging.md).

## Безопасность

- Валидация входа на границе (Pydantic/формы).
- SQL — параметризованно; пароли — `argon2`/`bcrypt`.
- Без `eval`/`exec`/`pickle` на недоверенных данных.
- Секреты/токены не в логах; внутренние ошибки не отдавать клиенту.
- Контроль: `ruff --select S`, `pip-audit`.

Подробно: [references/security.md](references/security.md) и навык `security`.

## Тестирование

- **pytest-функции и фикстуры**, не `unittest.TestCase`.
- Unit — только то, что недостижимо через BDD, плюс критичная логика.
- Без БД/HTTP/`TestClient`; зависимости — `AsyncMock`.
- Не тестируем Pydantic-валидацию, поля моделей, простой CRUD.

Подробно: `python-testing` (unit) и `pytest-bdd` (BDD).

## Документирование

- Google-style; документировать **«почему»**, не пересказывать сигнатуру.
- Матрица артефактов: тесты кратко; Django views подробно; FastAPI endpoints —
  English (ReDoc/OpenAPI); сервисы — «почему» + ограничения.
- Обязательно: публичные методы сервисов, сложные репозитории, валидаторы, сигналы.
- НЕ документировать: `__init__`, одно-строчные геттеры, простой CRUD.

Подробно: [references/documentation.md](references/documentation.md).

## Инструменты

```bash
uv run ruff check . && uv run ruff format . && uv run pyright && uv run pytest -v
```

- `uv` (пакеты/запуск), `ruff format` (вместо black), `pyright` (типы).
- Конфиг — в `pyproject.toml`; pre-commit: ruff + ruff-format + pyright.

Подробно: [references/tooling.md](references/tooling.md).

## Запрещённые паттерны

| ❌ | ✅ |
|---|---|
| Функции без аннотаций | полная типизация, `pyright` зелёный |
| Изменяемые дефолты (`def f(x=[])`) | `field(default_factory=list)` |
| `except Exception:` / голый `except` | конкретные исключения |
| `raise HTTPException` в сервисе | доменное исключение + маппинг |
| `print()` / f-строки в логах | `logger.info("event", key=value)` |
| `os.getenv()` вне `config.py` | `pydantic-settings` |
| Хардкод секретов/URL | переменные окружения |
| `time.sleep()`/`requests` в async | `asyncio.sleep`/`httpx` |
| `commit()` в репозитории | транзакция в сервисе |
| `Optional[X]`, `List[X]` | `X \| None`, `list[X]` |
| Ручное создание сервиса | DI (`__init__`/`Depends`) |
| `datetime.now()` без tz | `datetime.now(tz=UTC)` |

## Чек-лист code review

- [ ] Публичные сигнатуры типизированы, `pyright` зелёный.
- [ ] Слои не нарушены; нет SQL в presentation, логики в repository.
- [ ] Зависимости внедрены явно; fallback отсутствует.
- [ ] Транзакции в сервисе; репозиторий не коммитит; эффекты после коммита.
- [ ] Доменные исключения; маппинг централизован.
- [ ] Нет блокирующих вызовов в async.
- [ ] Секреты из окружения, не логируются.
- [ ] Логи структурированы, события `snake_case`.
- [ ] Unit не дублирует BDD; зависимости замоканы.
- [ ] Docstring там, где есть ценность.
- [ ] `ruff check`, `ruff format --check`, `pyright`, `pytest` проходят.

## Справочники

| Тема | Reference | Когда |
|---|---|---|
| Архитектура, слои, DI, транзакции | [references/architecture.md](references/architecture.md) | Проектирование сервисов/репозиториев |
| Типизация, Protocol, Pyright | [references/typing.md](references/typing.md) | Аннотации, интерфейсы, конфиг типов |
| Асинхронность | [references/async.md](references/async.md) | async/await, TaskGroup, timeout |
| Ошибки и логи | [references/errors-logging.md](references/errors-logging.md) | Исключения, handlers, structlog |
| Безопасность | [references/security.md](references/security.md) | Секреты, валидация, SQL, зависимости |
| Тестирование | `python-testing` + [references/testing.md](references/testing.md) | Unit/BDD, фикстуры, моки |
| Документирование | [references/documentation.md](references/documentation.md) | Docstrings, README, ADR |
| Инструменты | [references/tooling.md](references/tooling.md) | uv, ruff, pyright, pre-commit |

## Связанные навыки

- `python-testing` / `pytest-bdd` — тесты.
- `django` / `fastapi` — фреймворк-специфика.
- `security` — расширенный чек-лист безопасности.
- `documentation` — README/ADR.
- `python-audit` — оценка готовности.
