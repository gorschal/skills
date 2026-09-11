# Безопасность Python-кода

Безопасность — часть code review, а не отдельный этап. Ниже — обязательный
минимум; расширенный чек-лист — в навыке `security`.

## Секреты и конфигурация

- Секреты — только из окружения (`pydantic-settings`), не в коде и не в git.
- `.env` в `.gitignore`; в репозитории — `.env.example` без значений.
- Не логировать и не возвращать клиенту токены, пароли, `secret_key`, PII.
- Ротация секретов при утечке; разные секреты для dev/staging/prod.

```python
class Settings(BaseSettings):
    secret_key: SecretStr          # не отображается в repr/логах
    database_url: PostgresDsn
    model_config = SettingsConfigDict(env_file=".env")
```

## Валидация входа

- Валидировать на границе (Pydantic, Django Form), не доверять данным.
- Ограничивать длину, диапазоны, формат; `extra="forbid"` на command-схемах.
- Не использовать входные данные для путей/команд/запросов без санитайзинга.

```python
class UserCreate(BaseModel):
    email: EmailStr
    password: str = Field(min_length=8, max_length=128)
    model_config = ConfigDict(extra="forbid")
```

## SQL и ORM

- Только параметризованные запросы. Конкатенация значений — запрещена.

```python
# ✅ параметризация
await session.execute(text("SELECT * FROM users WHERE id = :id"), {"id": user_id})

# ❌ инъекция
await session.execute(text(f"SELECT * FROM users WHERE id = {user_id}"))
```

- ORM/Core с параметрами безопасны; опасны raw-строки с f-string.
- Минимальные права у пользователя БД; отдельные роли для чтения/записи.

## Пароли и криптография

- Хеширование: `argon2`/`bcrypt`/`pbkdf2` (Django/Passlib). Никакого `md5`/`sha1`.
- Не писать собственные криптоалгоритмы; использовать проверенные библиотеки.
- Для токенов — криптостойкий генератор (`secrets`), не `random`.

```python
import secrets
from passlib.context import CryptContext

pwd = CryptContext(schemes=["argon2"], deprecated="auto")
hashed = pwd.hash(password)
token = secrets.token_urlsafe(32)
```

## Небезопасная десериализация

- Запрещено: `pickle`, `eval`, `exec`, `marshal`, `yaml.load` (без `SafeLoader`)
  для недоверенных данных.
- JSON — через `json`/Pydantic. YAML — `yaml.safe_load`.

## Зависимости

- Регулярно обновлять; проверять уязвимости: `pip-audit`, `safety`.
- Фиксировать версии для приложений; диапазоны — для библиотек.
- Не тянуть пакеты без необходимости; проверять источник.

```bash
uv run pip-audit
uv run bandit -r src        # или ruff с правилами S
```

## Статический анализ безопасности

```toml
[tool.ruff.lint]
select = ["S"]              # flake8-bandit
```

Правила `S` ловят: `assert` в прод-коде, `subprocess` с `shell=True`, хардкод
паролей, небезопасные `hashlib`, `try/except/pass` и т.п.

## Веб/API-специфика (кратко)

- Никогда не отдавать внутренние ошибки и стектрейсы клиенту.
- CORS — только разрешённые origin; не `*` в проде с credentials.
- Ограничивать размер тела запроса и загрузок; валидировать MIME.
- Rate limiting на аутентификацию и чувствительные операции.
- Cookie: `HttpOnly`, `Secure`, `SameSite`; CSRF-защита для cookie-сессий.

## Антипаттерны

| ❌ | ✅ |
|---|---|
| Хардкод пароля/токена в коде | `Settings` из окружения |
| `f"SELECT ... {value}"` | параметризованный запрос |
| `pickle.loads(untrusted)` | JSON/Pydantic |
| `eval(user_input)` | безопасный парсер |
| `hashlib.md5(password)` | `argon2`/`bcrypt` |
| `random` для токенов | `secrets` |
| `shell=True` с вводом | список аргументов, `shell=False` |
| Логирование токенов | маскирование |

## Чек-лист

- [ ] Секретов нет в коде и git; `.env` игнорируется.
- [ ] Секреты не логируются и не отдаются клиенту.
- [ ] Вход валидируется; command-схемы `extra="forbid"`.
- [ ] SQL параметризован.
- [ ] Пароли хешируются современным алгоритмом.
- [ ] Нет `eval`/`exec`/`pickle` на недоверенных данных.
- [ ] `pip-audit`/`bandit`/`ruff S` без критичных находок.
- [ ] Общий обработчик не раскрывает стектрейс.
