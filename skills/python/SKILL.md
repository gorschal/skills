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
  version: "1.1.0"
  domain: language
  triggers: Python, typing, Pyright, mypy, asyncio, pytest, structlog, ruff, uv, pydantic, architecture, security
  role: specialist
  scope: implementation
  output-format: code
  related-skills: python-testing, django, fastapi, security, documentation, python-audit
---

# Python

Базовый навык для production-кода на Python 3.12+: слоистая архитектура,
полная типизация, async, доменные ошибки, structlog, безопасность, unit-тесты,
Google-docstrings и инструменты (`uv`, `ruff`, `pyright`).

> Этот навык — язык и общие практики. Фреймворк-специфика — в навыках
> `django` и `fastapi`; тестовая инфраструктура — в `python-testing`.

## Когда применять

- Написание/рефакторинг Python-кода: сервисы, репозитории, схемы, утилиты.
- Настройка типизации (`pyright`/`mypy`), линтинга (`ruff`), проекта (`uv`).
- Проектирование слоёв, DI, транзакций, обработки ошибок, логирования.
- Ревью Python-кода на типы, безопасность, тесты, docstring.
- НЕ применять для Django/FastAPI-специфики — переходить в `django`/`fastapi`.

## Ключевые принципы

1. **Код без аннотаций типов — брак.** Все аргументы и возвращаемые значения
   типизированы; проверка — `pyright` (strict).
2. **Unit-first тестирование.** Бизнес-логика проверяется изолированно, зависимости
   мокаются. Интеграционные/e2e-тесты — в отдельном QA-репозитории.
3. **Ошибки — доменными исключениями**, не кодами возврата. `raise low, catch high`.
4. **Границы слоёв неприкосновенны.** Роутер/вью не знает SQL; репозиторий не
   содержит бизнес-логики; сервис не знает про HTTP/`request`.
5. **Транзакции — только в сервисе.** Репозиторий не вызывает `commit`/`rollback`.
6. **Секреты — только из окружения.** Никакого хардкода и `os.getenv()` вне `config`.
7. **Логи — структурные** (structlog), события `snake_case` прошедшего времени,
   без `print` и f-строк.
8. **Не изобретать паттерны.** Следовать шаблонам репозитория; отклонение
   объяснять.

## Архитектура (слои)

| Слой                               | Ответственность                                             | Не имеет права                                 |
| ---------------------------------- | ----------------------------------------------------------- | ---------------------------------------------- |
| Presentation (router/view/handler) | вход/выход, валидация, один вызов сервиса                   | SQL, бизнес-логика, `try/except` бизнес-ошибок |
| Service                            | бизнес-логика, транзакции, оркестрация, доменные исключения | `request`/`response`, прямой SQL               |
| Repository                         | доступ к данным, только запросы                             | бизнес-логика, `commit`/`rollback`             |
| Schema/DTO                         | валидация и форма данных (Pydantic/dataclass/TypedDict)     | побочные эффекты                               |
| Domain/Config                      | конфиг, исключения, логирование, метаданные                 | прикладная логика                              |

- Зависимости передаются **явно через `__init__`** (или DI-фреймворк). Fallback
  на репозиторий (`self.repo = repo or Model.objects`) запрещён.
- Репозиторий типизируется `Protocol`; сервис принимает протокол, а не конкретный
  класс, — это делает моки простыми.
- Транзакция охватывает всю мутирующую операцию; побочные эффекты (письма,
  публикация задач) — после коммита (`transaction.on_commit` / outbox).

Подробно: [references/architecture.md](references/architecture.md).

## Типизация

```python
from collections.abc import Sequence, Mapping
from typing import Protocol

def process(items: Sequence[str], *, limit: int = 10) -> list[str]:
    """Принимает любую последовательность, возвращает новый список."""
    return [item.upper() for item in items[:limit]]

class UserRepo(Protocol):
    def get(self, user_id: int) -> "User | None": ...
```

- `X | None` вместо `Optional[X]`; `list[str]` вместо `List[str]`.
- `Sequence`/`Mapping` для чтения, `list`/`dict` для мутации.
- `TypedDict` — для строк БД и внешних payload; `Protocol` — для интерфейсов.
- `Literal`, `Final`, `TypeAlias`, `Self`, `@overload` — по назначению.
- `Any` допустим только на границе с нетипизированной библиотекой.

Подробно: [references/typing.md](references/typing.md).

## Асинхронность

| ❌ Запрещено          | ✅ Замена                    |
| --------------------- | ---------------------------- |
| `requests.get()`      | `httpx.AsyncClient`          |
| `time.sleep()`        | `await asyncio.sleep()`      |
| `psycopg2`, `sqlite3` | `asyncpg` + SQLAlchemy async |
| `open()`              | `aiofiles.open()`            |

- `async def` — если есть хотя бы один `await`; иначе обычный `def`.
- Блокирующий вызов в async-контексте — только через `run_in_threadpool`.
- Групповые операции — `asyncio.TaskGroup`; таймауты — `asyncio.timeout()`.
- Никаких «висящих» задач: фоновые задачи отслеживаются и отменяются на shutdown.

Подробно: [references/async.md](references/async.md).

## Обработка ошибок

```python
class DomainError(Exception):
    """Базовая ошибка домена."""

class UserNotFoundError(DomainError):
    def __init__(self, user_id: int) -> None:
        super().__init__(f"User {user_id} not found")

# сервис — выбрасывает доменное исключение
async def get_user(self, user_id: int) -> User:
    user = await self.repo.get(user_id)
    if user is None:
        raise UserNotFoundError(user_id)
    return user
```

- В сервисе — только доменные исключения. `HTTPException` запрещён.
- Маппинг на HTTP/транспорт — централизованно (global exception handler,
  middleware, ack/nack), не в бизнес-коде.
- Переиспользование причины: `raise NewError(...) from original`.
- Ловить исключение только чтобы: повторить, трансформировать, очистить ресурсы
  или добавить контекст. Иначе — пробрасывать.

Подробно: [references/errors-logging.md](references/errors-logging.md).

## Конфигурация и секреты

```python
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    database_url: str
    secret_key: str
    debug: bool = False
    model_config = SettingsConfigDict(env_file=".env")

settings = Settings()

def get_settings() -> Settings:
    return settings
```

- Всё окружение — через `Settings`; `os.getenv()` вне `config.py` запрещён.
- Секреты не логируются, не попадают в ответы и не коммитятся.
- `.env` в `.gitignore`; в репозитории — `.env.example` без значений.

## Логирование

```python
import structlog
logger = structlog.get_logger(__name__)

logger.info("user_created", user_id=user.id)
logger.exception("payment_failed", payment_id=str(payment_id))
```

| ✅                                        | ❌                            |
| ----------------------------------------- | ----------------------------- |
| `logger.info("user_created", user_id=id)` | `print(f"User {id} created")` |
| `logger.exception(...)` внутри `except`   | f-строки в логах              |
| события `snake_case` прошедшего времени   | «User Created»                |

- Контекст (`request_id`, `user_id`, `correlation_id`) — через
  `structlog.contextvars.bind_contextvars`; очищать в `finally`.

Подробно: [references/errors-logging.md](references/errors-logging.md).

## Безопасность

- Валидировать вход на границе (Pydantic/формы), не доверять данным.
- SQL — только параметризованно; конкатенация значений запрещена.
- Пароли — `bcrypt`/`argon2`/`pbkdf2`, никогда в открытом виде.
- Не использовать `eval`/`exec`/`pickle` для недоверенных данных.
- Секреты и токены не писать в логи; не отдавать внутренние ошибки клиенту.
- Статический контроль: `ruff` с правилами `S` (bandit), `pip-audit`/`safety`.

Подробно: [references/security.md](references/security.md).

## Тестирование

Политика монорепозитория: **BDD — основное сквозное покрытие**, unit-тесты —
дополнение для логики вне BDD и критичных веток. Unit не дублирует BDD.

```python
from unittest.mock import AsyncMock
import pytest

async def test_get_user_raises_when_missing() -> None:
    repo = AsyncMock()
    repo.get.return_value = None
    service = UserService(repo=repo)

    with pytest.raises(UserNotFoundError):
        await service.get_user(1)
```

- **pytest-функции и фикстуры**, не `unittest.TestCase`.
- Unit — только то, что недостижимо через BDD, плюс критичная логика.
- Без БД, HTTP и `TestClient`; зависимости замоканы (`AsyncMock`).
- Не тестируем Pydantic-валидацию, поля моделей, простой CRUD.

Подробно: [references/testing.md](references/testing.md).

## Документирование

```python
async def create_user(self, email: str, password: str) -> User:
    """Создаёт неактивного пользователя.

    Args:
        email: Адрес электронной почты (уникальный).
        password: Пароль в открытом виде (минимум 8 символов).

    Returns:
        User с is_active=False.

    Raises:
        UserAlreadyExistsError: Если email уже занят.

    Side Effects:
        Запись в БД, отправка welcome-email после коммита.
    """
```

- Google-style. Документировать **«почему»**, а не пересказывать сигнатуру.
- Детализация и язык — по матрице артефактов: тесты кратко; Django views
  подробно; FastAPI endpoints — English, как описание для ReDoc/OpenAPI;
  сервисы — «почему» + ограничения.
- Обязательно: публичные методы сервисов, сложные репозитории, кастомные
  валидаторы, сигналы, нетривиальные модели.
- НЕ документировать: `__init__`, одно-строчные геттеры/сеттеры, простой CRUD.

Подробно: [references/documentation.md](references/documentation.md).

## Инструменты

```bash
uv run ruff check .        # линт
uv run ruff format .       # формат
uv run pyright             # типы (strict)
uv run pytest -v           # тесты
```

- Пакеты и запуск — `uv`; формат — `ruff format` (вместо black); типы — `pyright`.
- Конфиг инструментов — в `pyproject.toml`.
- Pre-commit: `ruff` (fix) + `ruff-format` + `pyright`.

Подробно: [references/tooling.md](references/tooling.md).

## Запрещённые паттерны

| ❌ Запрещено                                     | ✅ Правильно                                             |
| ------------------------------------------------ | -------------------------------------------------------- |
| Функции без аннотаций типов                      | Полная типизация, `pyright` проходит                     |
| Изменяемые значения по умолчанию (`def f(x=[])`) | `field(default_factory=list)` / `None` + создание внутри |
| `except Exception:` / голый `except`             | Конкретные исключения                                    |
| `raise HTTPException` в сервисе                  | Доменное исключение + централизованный маппинг           |
| `print()` / f-строки в логах                     | `logger.info("event", key=value)`                        |
| `os.getenv()` вне `config.py`                    | `pydantic-settings` `Settings`                           |
| Хардкод секретов/URL                             | Переменные окружения                                     |
| `time.sleep()` / `requests` в async              | `asyncio.sleep` / `httpx.AsyncClient`                    |
| `commit()` в репозитории                         | Транзакция в сервисе                                     |
| `Optional[X]`, `List[X]` (старый стиль)          | `X \| None`, `list[X]`                                   |
| Ручное создание сервиса внутри сервиса/вью       | DI через `__init__`/`Depends`                            |
| `datetime.now()` без tz                          | `datetime.now(tz=UTC)`                                   |

## Чек-лист code review

- [ ] Все публичные функции/методы/атрибуты типизированы, `pyright` зелёный.
- [ ] Слои не нарушены: нет SQL в presentation, нет бизнес-логики в repository.
- [ ] Зависимости внедрены явно; fallback на репозиторий отсутствует.
- [ ] Транзакции в сервисе; репозиторий не коммитит.
- [ ] Побочные эффекты — после коммита.
- [ ] Доменные исключения; маппинг централизован.
- [ ] Нет блокирующих вызовов в async-коде.
- [ ] Секреты только из окружения, не логируются.
- [ ] Логи структурированы, события `snake_case`.
- [ ] Есть unit-тесты на бизнес-логику, зависимости замоканы.
- [ ] Docstring (Google) там, где есть ценность.
- [ ] `ruff check`, `ruff format --check`, `pyright`, `pytest` проходят.

## Справочники

| Тема                                   | Reference                                                    | Загружать когда                                         |
| -------------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------- |
| Архитектура, слои, DI, транзакции      | [references/architecture.md](references/architecture.md)     | Проектирование сервисов/репозиториев, структура проекта |
| Типизация, generics, Protocol, Pyright | [references/typing.md](references/typing.md)                 | Аннотации, интерфейсы, конфиг проверки типов            |
| Асинхронность                          | [references/async.md](references/async.md)                   | async/await, TaskGroup, timeout, sync↔async             |
| Ошибки и логирование                   | [references/errors-logging.md](references/errors-logging.md) | Исключения, global handlers, structlog                  |
| Безопасность                           | [references/security.md](references/security.md)             | Секреты, валидация, SQL, зависимости                    |
| Тестирование                           | [references/testing.md](references/testing.md)               | pytest, fixtures, mocks, parametrize                    |
| Документирование                       | [references/documentation.md](references/documentation.md)   | Docstrings, README, ADR                                 |
| Инструменты                            | [references/tooling.md](references/tooling.md)               | uv, ruff, pyright, pre-commit, CI                       |

## Связанные навыки

- `python-testing` — углублённая тестовая инфраструктура и политика.
- `django` / `fastapi` — фреймворк-специфика (ORM, DI, transactions API).
- `security` — расширенный чек-лист безопасности.
- `documentation` — README/ADR/AGENTS.md.
- `python-audit` — оценка готовности проекта.
