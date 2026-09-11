# Pydantic v2

Схемы — только Pydantic. ORM-модели и API-схемы — разные классы.

## Обязательные три схемы

```python
from datetime import datetime
from uuid import UUID
from pydantic import BaseModel, ConfigDict, EmailStr, Field

class UserCreate(BaseModel):
    email: EmailStr
    password: str = Field(min_length=8, max_length=128)
    model_config = ConfigDict(extra="forbid")

class UserOut(BaseModel):
    id: int
    email: EmailStr
    created_at: datetime
    model_config = ConfigDict(from_attributes=True)

class UserUpdate(BaseModel):
    email: EmailStr | None = None
    is_active: bool | None = None
    model_config = ConfigDict(extra="forbid")
```

- `Create`/`Update` — `extra="forbid"` (запрет лишних полей).
- `Out` — `from_attributes=True` для валидации из ORM/`TypedDict`.
- `Update` — все поля опциональны.

## Ограничения полей

```python
from typing import Annotated
from pydantic import Field

UserEmail = Annotated[EmailStr, Field(min_length=5)]
Amount = Annotated[int, Field(gt=0)]

class PaymentCreate(BaseModel):
    amount: Amount
    model_config = ConfigDict(extra="forbid")
```

- Ограничения — через `Field`/`Annotated`, не в валидаторах без нужды.
- Изменяемые значения по умолчанию — только `default_factory`.

## Валидаторы

```python
from pydantic import field_validator, model_validator
from typing import Self

class UserCreate(BaseModel):
    email: EmailStr
    password: str

    @field_validator("password")
    @classmethod
    def strong_enough(cls, v: str) -> str:
        if len(v) < 8:
            raise ValueError("password too short")
        return v

class OrderCreate(BaseModel):
    items: list[OrderItem]
    total: int

    @model_validator(mode="after")
    def check_total(self) -> Self:
        if self.total != sum(i.price * i.quantity for i in self.items):
            raise ValueError("total mismatch")
        return self
```

- `@field_validator` — одно поле; `@model_validator(mode="after")` — кросс-поля.
- Валидаторы быстрые и без побочных эффектов/IO.

## Сериализация

```python
class User(BaseModel):
    id: int
    email: EmailStr
    password: str = Field(exclude=True)   # никогда не сериализуется
```

- `Field(exclude=True)` для секретов.
- Вывод: `.model_dump()` / `.model_dump_json()` (не `.dict()`).

## Настройки

```python
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    database_url: str
    secret_key: SecretStr
    debug: bool = False
    model_config = SettingsConfigDict(env_file=".env", case_sensitive=True)

settings = Settings()
```

- `SecretStr` для секретов (не отображается в repr/логах).
- Падение на старте при отсутствии обязательной переменной — это хорошо.

## Подводные камни

- **Silent coercion**: Pydantic приводит типы. Где нужна строгость — `StrictInt`/`StrictStr`.
- **Изменяемые дефолты**: только `Field(default_factory=list)`.
- **Тяжёлые валидаторы**: без IO и тяжёлых вычислений.
- **`from_attributes` только на выходе** — не на входных схемах.

## Pydantic v1 → v2

| v1                     | v2                               |
| ---------------------- | -------------------------------- |
| `@validator`           | `@field_validator`               |
| `@root_validator`      | `@model_validator`               |
| `class Config`         | `model_config = ConfigDict(...)` |
| `orm_mode = True`      | `from_attributes = True`         |
| `Optional[X]`          | `X \| None`                      |
| `.dict()`              | `.model_dump()`                  |
| `.parse_obj()`         | `.model_validate()`              |
| `allow_mutation=False` | `frozen=True`                    |

## Антипаттерны

| ❌                              | ✅                                                |
| ------------------------------- | ------------------------------------------------- |
| `class Config: orm_mode = True` | `model_config = ConfigDict(from_attributes=True)` |
| `@validator`                    | `@field_validator`                                |
| `.dict()` / `.parse_obj()`      | `.model_dump()` / `.model_validate()`             |
| `extra` не ограничен на входе   | `extra="forbid"`                                  |
| `def f(tags=[])`                | `Field(default_factory=list)`                     |
| Секрет в схеме вывода           | `Field(exclude=True)`                             |
| Одна схема на чтение и запись   | Раздельные Create/Out/Update                      |

## Чек-лист

- [ ] Раздельные Create/Out/Update схемы.
- [ ] `extra="forbid"` на входных схемах.
- [ ] `from_attributes=True` только на выходных.
- [ ] Ограничения через `Field`/`Annotated`.
- [ ] Нет изменяемых дефолтов.
- [ ] Секреты исключены из сериализации.
- [ ] Только синтаксис Pydantic v2.
