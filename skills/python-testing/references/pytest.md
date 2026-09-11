# pytest: структура, фикстуры, покрытие

## Структура теста

```python
import pytest

def test_user_creation() -> None:
    user = User(id=1, name="Alice")
    assert user.name == "Alice"

def test_invalid_email() -> None:
    with pytest.raises(ValueError, match="Invalid email"):
        User(id=1, email="invalid")
```

- pytest-функции, **не** `unittest.TestCase` (иначе теряются фикстуры и
  `parametrize`).
- Имя — описание поведения: `test_rejects_expired_token`.
- Один тест — одно поведение.

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

@pytest.fixture
def db_session() -> Iterator[Session]:
    session = create_session()
    yield session
    session.rollback()
    session.close()
```

- `conftest.py` — общие фикстуры для пакета/каталога.
- `scope="function"` по умолчанию (изоляция); `session` — только для дорогих
  неизменяемых ресурсов.
- Очистка — `yield` + teardown.
- `autouse=True` — только для сквозных вещей (например, сброс кэша).

### Фабрики

```python
@pytest.fixture
def user_factory():
    def _make(name: str = "Test", active: bool = True) -> User:
        return User(name=name, active=active)
    return _make

def test_names(user_factory) -> None:
    assert user_factory("Alice").name == "Alice"
```

## Параметризация

```python
@pytest.mark.parametrize(
    "email,valid",
    [("a@b.c", True), ("invalid", False), ("@b.c", False)],
    ids=["valid", "no_at", "no_domain"],
)
def test_email_validation(email: str, valid: bool) -> None:
    assert is_valid_email(email) is valid
```

- `ids` — читаемые имена кейсов.
- Несколько `parametrize` декораторов комбинируются.

## Маркеры

```python
@pytest.mark.slow
def test_slow() -> None: ...

@pytest.mark.skip(reason="not implemented")
def test_future() -> None: ...

@pytest.mark.xfail(reason="known bug #123")
def test_known_bug() -> None: ...
```

```toml
[tool.pytest.ini_options]
addopts = ["-ra", "--strict-markers"]
markers = ["slow: медленные", "integration: интеграционные"]
```

`--strict-markers` — незарегистрированные маркеры ломают прогон.

## Покрытие

```toml
[tool.pytest.ini_options]
addopts = ["-ra", "--strict-markers", "--cov=src", "--cov-report=term-missing"]
testpaths = ["tests"]

[tool.coverage.run]
source = ["src"]
branch = true

[tool.coverage.report]
exclude_lines = ["pragma: no cover", "if TYPE_CHECKING:", "raise NotImplementedError"]
```

- `--cov-fail-under=N` — если нужен гейт (осознанно).
- Покрытие — ориентир; критичные ветки обязательны.

## Организация

```
tests/
├── conftest.py          # общие фикстуры
├── unit/
│   └── test_services.py
└── integration/         # если есть тонкий слой (иначе — BDD/QA)
```

## Антипаттерны

| ❌ | ✅ |
|---|---|
| `unittest.TestCase` | pytest-функции/фикстуры |
| Тест зависит от порядка | изоляция |
| Общий мутабельный state | свежие инстансы/фикстуры |
| `scope="session"` для изменяемого | `function` |
| Нет teardown для ресурсов | `yield` + очистка |
| `assert result` | `assert result == expected` |

## Чек-лист

- [ ] pytest-функции/фикстуры; имена — описания поведения.
- [ ] Фикстуры изолированы, с очисткой.
- [ ] `parametrize` вместо копипасты.
- [ ] Маркеры зарегистрированы (`--strict-markers`).
- [ ] Покрытие настроено; критичные ветки покрыты.
