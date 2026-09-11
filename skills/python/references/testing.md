# Тестирование (BDD-first, unit для пробелов)

Политика монорепозитория: **основное сквозное покрытие — BDD** (`pytest-bdd`).
Unit-тесты — дополнение: берут логику, недостижимую через BDD, и критичные
ветки. Цель — не дублировать покрытие и не тратить время на избыточные unit-тесты.

## Тестовая пирамида монорепозитория

| Уровень | Роль | Покрытие |
|---|---|---|
| BDD (`pytest-bdd`) | основное сквозное тестирование всей логики | вся бизнес-логика end-to-end |
| Unit (pytest) | дополнение: логика вне BDD + критичные ветки | точечно, критичное |
| Интеграционные/e2e | вне основного репозитория (QA) | по необходимости |

**Ключевое правило: unit не дублирует BDD.** Если сценарий проверяется через
BDD — отдельный unit-тест на ту же логику не пишем.

## Что попадает в unit-тесты

1. Логика, недостижимая через BDD: ветки ошибок, которые сложно/дорого
   воспроизвести снаружи, внутренние расчёты, граничные условия.
2. Критичная бизнес-логика, где нужна быстрая обратная связь.
3. Чистые функции и утилиты без внешних зависимостей.

## Что НЕ тестируем unit-тестами

- То, что уже покрыто BDD-сценарием.
- Pydantic-валидацию, поля моделей, простой CRUD.
- HTTP-статусы, роутеры, middleware (уровень BDD).
- Геттеры/сеттеры, `__str__`.

**Запрещено в unit:** `TestClient`, реальный HTTP, фикстуры БД,
`obj.save()`/`QuerySet.create()`.

## Стиль: pytest, не unittest

- pytest-функции и фикстуры, **не** `unittest.TestCase`.
- Это сохраняет `parametrize`, fixtures и лаконичные ассерты.

```python
from unittest.mock import AsyncMock
import pytest

async def test_create_user_raises_when_email_taken() -> None:
    repo = AsyncMock()
    repo.exists_by_email.return_value = True
    service = UserService(repo=repo)

    with pytest.raises(UserAlreadyExistsError):
        await service.create_user("taken@example.com", "password123")

    repo.exists_by_email.assert_awaited_once_with("taken@example.com")
```

## Фикстуры

```python
import pytest
from unittest.mock import AsyncMock

@pytest.fixture
def repo() -> AsyncMock:
    return AsyncMock()

@pytest.fixture
def service(repo: AsyncMock) -> UserService:
    return UserService(repo=repo)

async def test_get_user(service: UserService, repo: AsyncMock) -> None:
    repo.get.return_value = {"id": 1, "email": "a@b.c"}
    user = await service.get_user(1)
    assert user.email == "a@b.c"
```

- `conftest.py` — общие фикстуры; `scope="function"` по умолчанию.
- Ресурсы с очисткой — через `yield` + teardown.
- Фабрики (`user_factory`) — для вариативных данных.

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

## Мокирование исключений и side effects

```python
repo.get.side_effect = [None, {"id": 1}]     # 1-й вызов None, 2-й — запись
repo.save.side_effect = DatabaseError("boom")

with pytest.raises(ExternalServiceError):
    await service.process(1)
```

## Async-тесты

```toml
[tool.pytest.ini_options]
asyncio_mode = "auto"
```

```python
async def test_idempotent_processing(repo: AsyncMock) -> None:
    repo.is_processed.return_value = True
    service = PaymentService(repo=repo)

    with pytest.raises(PaymentAlreadyProcessedError):
        await service.process(Payment(id=1))
```

## Что проверять в сервисе

- Ветвления бизнес-правил (границы, нули, отрицательные значения).
- Доменные исключения в каждой ошибочной ситуации.
- Вызовы зависимостей с правильными аргументами.
- Идемпотентность: повторный вызов не создаёт дубль.
- Побочные эффекты запланированы после коммита.

## Покрытие

```toml
[tool.pytest.ini_options]
addopts = ["-ra", "--strict-markers", "--cov=src", "--cov-report=term-missing"]
testpaths = ["tests"]
```

- Покрытие — ориентир, не самоцель: полное покрытие даёт BDD, unit закрывает
  пробелы и критичные ветки.
- Не гнаться за 100% на тривиальном коде.

## Антипаттерны

| ❌ | ✅ |
|---|---|
| Дублировать BDD-сценарий unit-тестом | unit только для пробелов BDD |
| `unittest.TestCase` | pytest-функции и фикстуры |
| `TestClient`/реальная БД | моки зависимостей |
| Проверка Pydantic-валидации | это делает BDD |
| Тест зависит от другого теста | полная изоляция |
| Один тест — десять сценариев | `parametrize` |
| `assert mock.called` без аргументов | `assert_awaited_once_with(...)` |

## Чек-лист

- [ ] Unit-тесты не дублируют BDD-покрытие.
- [ ] Покрыта логика вне BDD и критичные ветки.
- [ ] Используются pytest-функции/фикстуры, не `unittest.TestCase`.
- [ ] Зависимости замоканы (`AsyncMock` для async).
- [ ] Нет `TestClient`, реального HTTP, БД.
- [ ] Тесты изолированы и детерминированы.
