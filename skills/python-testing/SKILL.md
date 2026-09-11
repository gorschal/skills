---
name: python-testing
description: >
  Use when writing or reviewing pytest unit/integration tests in Python: test
  structure, fixtures, parametrize, mocking (Mock/AsyncMock/patch), async tests,
  coverage, factories, and test-quality anti-patterns. Триггеры: pytest, unit
  test, fixture, parametrize, mock, AsyncMock, patch, coverage, factory_boy,
  hypothesis, flaky test, test isolation. Для BDD/Gherkin — навык pytest-bdd.
license: MIT
compatibility: opencode
metadata:
  version: "1.0.0"
  domain: testing
  triggers: pytest, unit test, fixture, parametrize, mock, AsyncMock, coverage, factory_boy, flaky test
  role: specialist
  scope: implementation
  output-format: code
  related-skills: python, pytest-bdd, django, fastapi, faststream, aiogram
---

# Python Testing

Unit/integration-тесты на pytest. В монорепозитории **BDD — основное сквозное
покрытие**; unit-тесты дополняют: логика вне BDD и критичные ветки.

> BDD/Gherkin/Page Objects — навык `pytest-bdd`.

## Когда применять

- Написание unit-тестов сервисов, утилит, чистых функций.
- Настройка pytest: фикстуры, параметризация, покрытие, маркеры.
- Ревью тестов на качество (моки, изоляция, флаки).

## Политика: unit ↔ BDD

| Уровень | Что проверяет | Инструмент |
|---|---|---|
| BDD | сквозные сценарии всей логики (основное покрытие) | `pytest-bdd` |
| Unit | логика вне BDD + критичные ветки | `pytest` + моки |

- Unit **не дублирует** BDD.
- Unit берёт: ветки ошибок, недостижимые через BDD; граничные условия; критичные
  расчёты; чистые функции.
- Интеграционные/e2e — в BDD/QA, не в unit.

## Ключевые принципы

1. **Тестировать поведение, а не моки.** Проверять результат/эффект, а не только
   `assert_called_once`.
2. **Один тест — одно поведение.**
3. **Полная изоляция:** тест не зависит от порядка и других тестов.
4. **Специфичные ассерты** (`== 90`), не `assert result`.
5. **Happy path + ошибки/границы** (пусто, null, границы).
6. **Мокать внешнее** (сеть, БД, время), не внутреннюю логику.
7. **Флаки-тесты чинить**, не «перезапускать до зелёного».

## Структура и фикстуры

```python
import pytest
from unittest.mock import AsyncMock

@pytest.fixture
def repo() -> AsyncMock:
    return AsyncMock()

@pytest.fixture
def service(repo: AsyncMock) -> UserService:
    return UserService(repo=repo)

async def test_get_user_raises_when_missing(service: UserService, repo: AsyncMock) -> None:
    repo.get.return_value = None

    with pytest.raises(UserNotFoundError):
        await service.get_user(1)
```

- pytest-функции и фикстуры, **не** `unittest.TestCase`.
- `conftest.py` — общие фикстуры; `scope="function"` по умолчанию.
- Очистка — `yield` + teardown; фабрики — для вариативных данных.

Подробно: [references/pytest.md](references/pytest.md).

## Параметризация

```python
@pytest.mark.parametrize(
    "amount,expected",
    [(0, False), (100, True), (-1, False)],
    ids=["zero", "valid", "negative"],
)
def test_is_valid_amount(amount: int, expected: bool) -> None:
    assert is_valid_amount(amount) is expected
```

## Мокирование

```python
from unittest.mock import Mock, AsyncMock, patch

repo = AsyncMock()
repo.get.return_value = {"id": 1}

repo.get.side_effect = [None, {"id": 1}]
repo.save.side_effect = DatabaseError("boom")

with patch("app.services.transaction.atomic"):
    ...
```

- Async-методы — `AsyncMock`; sync — `Mock`.
- Мокать на границе (внешние сервисы), а не внутренние детали.
- Моки полные (все поля, которые читает код) — использовать фабрики.
- Не тестировать «что мок вызван» без проверки поведения.

Подробно: [references/mocking.md](references/mocking.md).

## Async-тесты

```toml
[tool.pytest.ini_options]
asyncio_mode = "auto"
```

```python
async def test_concurrent(repo: AsyncMock) -> None:
    repo.is_processed.return_value = True
    service = PaymentService(repo=repo)

    with pytest.raises(PaymentAlreadyProcessedError):
        await service.process(Payment(id=1))
```

- `pytest-asyncio`; async-фикстуры с `AsyncMock`/`yield`.
- Не вызывать реальный event loop-блокирующий код.

Подробно: [references/async.md](references/async.md).

## Покрытие

```toml
[tool.pytest.ini_options]
addopts = ["-ra", "--strict-markers", "--cov=src", "--cov-report=term-missing"]
testpaths = ["tests"]
```

- Покрытие — ориентир: полное покрытие даёт BDD, unit закрывает пробелы.
- Критичные ветки и обработчики ошибок обязательны.
- Не гнаться за 100% на тривиальном коде.

## Качество тестов

- Тестировать наблюдаемое поведение, не реализацию.
- Избегать тест-онли методов в продакшене; свежие инстансы вместо `_reset()`.
- Не мокать всё подряд — реальные зависимости, где возможно.
- Флаки: изолировать, чинить причину (порядок, async, время, сеть).

Подробно: [references/quality.md](references/quality.md).

## Запрещённые паттерны

| ❌ Запрещено | ✅ Правильно |
|---|---|
| `assert mock.called` без проверки результата | ассерт на поведение + при необходимости аргументы |
| Дублирование BDD-сценария unit-тестом | unit только для пробелов BDD |
| `unittest.TestCase` | pytest-функции/фикстуры |
| Тест зависит от порядка | полная изоляция |
| Реальные сеть/БД/время в unit | моки/`freeze_time`/фейки |
| Неполные моки | фабрики, полный контракт |
| Тест-онли методы в коде | тестовые утилиты, свежие инстансы |
| `assert result` без конкретики | `assert result == expected` |
| Игнор флаки | починить/изолировать |
| `print` вместо ассертов | явные ассерты |

## Чек-лист

- [ ] Unit не дублирует BDD; покрыта логика вне BDD и критичные ветки.
- [ ] Тесты проверяют поведение, а не только моки.
- [ ] pytest-функции/фикстуры, не `unittest.TestCase`.
- [ ] Полная изоляция; тесты не зависят от порядка.
- [ ] Async — `AsyncMock`/`pytest-asyncio`.
- [ ] Happy path + ошибки/границы.
- [ ] Ассерты специфичны; флаки не игнорируются.
- [ ] Покрытие настроено; критичные ветки покрыты.

## Справочники

| Тема | Reference | Загружать когда |
|---|---|---|
| pytest, фикстуры, покрытие | [references/pytest.md](references/pytest.md) | Структура тестов, фикстуры, parametrize |
| Мокирование | [references/mocking.md](references/mocking.md) | Mock/AsyncMock/patch, стратегия моков |
| Async | [references/async.md](references/async.md) | pytest-asyncio, async-фикстуры |
| Качество и антипаттерны | [references/quality.md](references/quality.md) | Ревью тестов, флаки, mock-антипаттерны |

## Связанные навыки

- `pytest-bdd` — BDD/Gherkin, Page Objects, отчёты.
- `python` — общие практики, docstrings, инструменты.
- `django` / `fastapi` / `faststream` / `aiogram` — тестируемые системы.
