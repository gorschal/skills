# Типизация

Полная типизация — обязательна. Код без аннотаций считается браком.
Проверка: `pyright` (strict); `mypy` — опционально для совместимости.

## Базовые аннотации

```python
from collections.abc import Sequence, Mapping
from typing import Any

def process_user(name: str, age: int, active: bool = True) -> dict[str, Any]:
    return {"name": name, "age": age, "active": active}

# Современный union
def find_user(user_id: int | str) -> dict[str, Any] | None:
    return {"id": user_id} if isinstance(user_id, int) else None

# Принимаем широко, возвращаем конкретно
def process_items(items: Sequence[str]) -> list[str]:
    return [item.upper() for item in items]

def merge(base: Mapping[str, int], override: dict[str, int]) -> dict[str, int]:
    return {**base, **override}
```

- `X | None` вместо `Optional[X]`; `list[X]`/`dict[K, V]` вместо `List`/`Dict`.
- `Sequence`/`Mapping`/`Iterable` — для чтения; `list`/`dict` — когда мутируем.
- `Any` — только на границе с нетипизированной библиотекой, не внутри кода.

## Generics

```python
from typing import TypeVar
from collections.abc import Sequence

T = TypeVar("T")

def first(items: Sequence[T]) -> T | None:
    return items[0] if items else None

class Cache[K, V]:                    # синтаксис 3.12
    def __init__(self) -> None:
        self._data: dict[K, V] = {}

    def get(self, key: K) -> V | None:
        return self._data.get(key)
```

## Protocol — структурная типизация

Интерфейс без наследования: удобно для репозиториев и моков.

```python
from typing import Protocol, runtime_checkable

class UserRepository(Protocol):
    def get(self, user_id: int) -> "User | None": ...
    def save(self, user: "User") -> None: ...

@runtime_checkable
class Closeable(Protocol):
    def close(self) -> None: ...
```

## TypedDict, Literal, Final, TypeAlias

```python
from typing import Literal, TypeAlias, TypedDict, NotRequired, Final

class UserRow(TypedDict):
    id: int
    email: str
    name: NotRequired[str]

Mode: TypeAlias = Literal["r", "w", "a"]
MAX_RETRIES: Final = 3

def open_file(path: str, mode: Mode) -> None: ...
```

- `TypedDict` — для строк БД и внешних payload.
- `Literal` — вместо «магических» строк; проверяется статически.
- `Final` — для констант.

## Self, overload, assert_never

```python
from typing import Self, overload, assert_never

class QueryBuilder:
    def where(self, cond: str) -> Self:
        self._conds.append(cond)
        return self

@overload
def parse(data: str) -> str: ...
@overload
def parse(data: int) -> int: ...
def parse(data: str | int) -> str | int:
    return data.upper() if isinstance(data, str) else data * 2

def handle(mode: Literal["read", "write"]) -> str:
    if mode == "read":
        return "r"
    if mode == "write":
        return "w"
    assert_never(mode)      # исчерпывающая проверка
```

## Сужение типов (narrowing)

```python
def render(value: int | str | None) -> str:
    if value is None:
        return ""
    if isinstance(value, int):
        return str(value)
    return value.upper()      # здесь value: str
```

## Конфигурация pyright (pyproject.toml)

```toml
[tool.pyright]
pythonVersion = "3.12"
typeCheckingMode = "strict"
reportMissingImports = true
reportUnusedImport = true
```

Запуск: `uv run pyright`. Любая ошибка типов должна быть устранена до сдачи.

## Конфигурация mypy (если нужен)

```toml
[tool.mypy]
python_version = "3.12"
strict = true
warn_return_any = true
disallow_untyped_defs = true

[[tool.mypy.overrides]]
module = "third_party.*"
ignore_missing_imports = true
```

## Антипаттерны

| ❌                                     | ✅                                                     |
| -------------------------------------- | ------------------------------------------------------ |
| `def f(x, y):`                         | `def f(x: int, y: str) -> None:`                       |
| `Optional[X]`, `List[X]`, `Dict[K, V]` | `X \| None`, `list[X]`, `dict[K, V]`                   |
| `Any` везде                            | конкретные типы, `object`/`Protocol` при необходимости |
| `# type: ignore` без причины           | исправить или указать код правила                      |
| Возврат `dict` вместо модели           | `TypedDict`/Pydantic/dataclass                         |
| `cast` без проверки                    | сужение через `isinstance`                             |

## Чек-лист

- [ ] Все аргументы и возвраты типизированы.
- [ ] Публичные атрибуты класса типизированы.
- [ ] Нет `Any` вне границ.
- [ ] `pyright` (strict) проходит без ошибок.
- [ ] `# type: ignore` — только с кодом и объяснением.
