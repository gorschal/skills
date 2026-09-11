# Тестирование FastAPI

Политика монорепозитория: **BDD — основное сквозное покрытие**; unit-тесты —
дополнение для логики вне BDD и критичных веток. Unit не дублирует BDD.
HTTP-слой, статус-коды и DI-обвязка проверяются через BDD (`pytest-bdd`).

## Границы

| Уровень | Что проверяет |
|---|---|
| BDD (`pytest-bdd`) | сквозные сценарии всей логики (основное покрытие) |
| Unit (pytest) | логика сервисов вне BDD + критичные ветки |

## Что тестируем unit-тестами

- Бизнес-логику сервисов (ветвления, расчёты, границы).
- Маппинг доменных исключений.
- Логику, недостижимую через BDD.

## Что НЕ тестируем

- То, что покрыто BDD-сценарием.
- Pydantic-валидацию, простой CRUD.
- HTTP-статусы, роутеры, middleware, DI-обвязку (BDD).
- Репозитории (сложные JOIN/CTE — контрактные тесты/QA).

**Запрещено в unit:** `TestClient`, `httpx.AsyncClient` по приложению, реальная БД.

## Стиль: pytest + AsyncMock

- pytest-функции и фикстуры, не `unittest.TestCase`.
- Зависимости — `AsyncMock` (async-методы).

```python
from unittest.mock import AsyncMock
from uuid import UUID
import pytest

async def test_get_payment_raises_when_missing() -> None:
    repo = AsyncMock()
    repo.get_with_account.return_value = None
    service = PaymentService(repo=repo)

    with pytest.raises(PaymentNotFoundError):
        await service.get_payment(UUID(int=1))

    repo.get_with_account.assert_awaited_once()
```

## Успешный путь

```python
async def test_get_payment_returns_schema() -> None:
    repo = AsyncMock()
    repo.get_with_account.return_value = {
        "id": UUID(int=1),
        "account_id": UUID(int=2),
        "amount": 100,
    }
    service = PaymentService(repo=repo)

    result = await service.get_payment(UUID(int=1))

    assert result.amount == 100
```

## Фикстуры

```python
import pytest
from unittest.mock import AsyncMock

@pytest.fixture
def payment_repo() -> AsyncMock:
    return AsyncMock()

@pytest.fixture
def payment_service(payment_repo: AsyncMock) -> PaymentService:
    return PaymentService(repo=payment_repo)
```

- `conftest.py` — общие фикстуры; `scope="function"` по умолчанию.
- Ресурсы с очисткой — `yield` + teardown.

## Параметризация

```python
@pytest.mark.parametrize(
    "amount,valid",
    [(0, False), (1, True), (-5, False)],
    ids=["zero", "positive", "negative"],
)
def test_amount_validation(amount: int, valid: bool) -> None:
    if valid:
        PaymentCreate(account_id=UUID(int=1), amount=amount)
    else:
        with pytest.raises(ValidationError):
            PaymentCreate(account_id=UUID(int=1), amount=amount)
```

## Антипаттерны

| ❌ | ✅ |
|---|---|
| Дублировать BDD-сценарий unit-тестом | unit только для пробелов BDD |
| `TestClient`/`httpx.AsyncClient` | `AsyncMock` сервисов/репозиториев |
| Реальная БД в unit | моки |
| Проверка Pydantic-валидации | это BDD |
| `unittest.TestCase` | pytest-функции |
| `assert mock.called` | `assert_awaited_once_with(...)` |

## Чек-лист

- [ ] Unit-тесты не дублируют BDD.
- [ ] Покрыта логика вне BDD и критичные ветки.
- [ ] pytest-функции/фикстуры, не `unittest.TestCase`.
- [ ] Зависимости — `AsyncMock`; нет реальной БД.
- [ ] Нет `TestClient`/`httpx.AsyncClient` по приложению.
- [ ] Тесты изолированы и детерминированы.
