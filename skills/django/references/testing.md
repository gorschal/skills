# Тестирование Django

Политика монорепозитория: **BDD — основное сквозное покрытие**; unit-тесты —
дополнение для логики вне BDD и критичных веток. Unit не дублирует BDD.
Подробнее о BDD — в навыке `pytest-bdd`.

## Границы

| Уровень | Что проверяет |
|---|---|
| BDD (`pytest-bdd`) | сквозные сценарии всей логики (основное покрытие) |
| Unit (pytest) | логика вне BDD + критичные ветки |

## Что тестируем unit-тестами

- Бизнес-логику сервисов (ветвления, расчёты, границы).
- Тела сигналов и Django Tasks.
- Сложные кастомные валидаторы.
- Логику, недостижимую через BDD.

## Что НЕ тестируем

- То, что покрыто BDD-сценарием.
- Поля моделей (`auto_now_add`, `unique=True`), `__str__`.
- Стандартную валидацию форм.
- HTTP/views, коды ответов (уровень BDD).

**Запрещено в unit:** `TestClient`, `self.client`, реальный HTTP, фикстуры БД,
`obj.save()`/`QuerySet.create()`.

## Стиль: pytest, не unittest

- pytest-функции и фикстуры, **не** `unittest.TestCase`/`django.test.TestCase`.
- Репозитории — `Mock`; транзакции — патчатся (unit не трогает БД).

```python
from unittest.mock import Mock, patch
import pytest

@patch("app.services.transaction.on_commit")
@patch("app.services.transaction.atomic")
def test_create_user_raises_when_email_taken(mock_atomic, mock_on_commit) -> None:
    mock_atomic.return_value.__enter__ = Mock()
    mock_atomic.return_value.__exit__ = Mock(return_value=False)
    repo = Mock()
    repo.filter.return_value.exists.return_value = True
    service = UserService(user_repo=repo)

    with pytest.raises(UserAlreadyExistsError):
        service.create_user("taken@example.com", "password123")

    repo.filter.assert_called_once_with(email="taken@example.com")
```

> Если сервис использует `transaction.atomic()`/`on_commit`, их нужно патчить —
> иначе unit-тест обратится к БД. Это следствие правила «всё сохранение через
> репозиторий»: сервис не вызывает `obj.save()` напрямую.

## Успешный путь с моками

```python
def test_create_user_returns_user() -> None:
    repo = Mock()
    repo.filter.return_value.exists.return_value = False
    repo.create_user.return_value = Mock(id=1, email="new@example.com")
    service = UserService(user_repo=repo)

    with patch("app.services.transaction.atomic"), patch("app.services.transaction.on_commit"):
        user = service.create_user("new@example.com", "password123")

    assert user.email == "new@example.com"
    repo.create_user.assert_called_once_with("new@example.com", "password123")
```

## Фикстуры

```python
import pytest
from unittest.mock import Mock

@pytest.fixture
def user_repo() -> Mock:
    repo = Mock()
    repo.filter.return_value.exists.return_value = False
    return repo

@pytest.fixture
def user_service(user_repo: Mock) -> UserService:
    return UserService(user_repo=user_repo)
```

- `conftest.py` — общие фикстуры; `scope="function"` по умолчанию.
- Ресурсы с очисткой — `yield` + teardown.

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

## Антипаттерны

| ❌ | ✅ |
|---|---|
| Дублировать BDD-сценарий unit-тестом | unit только для пробелов BDD |
| `django.test.TestCase`/`APITestCase` | pytest-функции + `Mock` |
| `self.client.get/post` | BDD-сценарий |
| Фикстуры БД / `objects.create()` | `Mock` репозитория |
| Проверка `__str__`/полей модели | не тестировать |
| `assert mock.called` | `assert_called_once_with(...)` |

## Чек-лист

- [ ] Unit-тесты не дублируют BDD.
- [ ] Покрыта логика вне BDD и критичные ветки.
- [ ] pytest-функции/фикстуры, не `unittest.TestCase`.
- [ ] Репозитории замоканы (`Mock`), транзакции патчатся.
- [ ] Нет `TestClient`, `self.client`, БД.
- [ ] Тесты изолированы и детерминированы.
