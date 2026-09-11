# Безопасность FastAPI

Расширенный чек-лист — в навыке `security`.

## Пароли

```python
from passlib.context import CryptContext

pwd = CryptContext(schemes=["argon2", "bcrypt"], deprecated="auto")

hashed = pwd.hash(password)
ok = pwd.verify(password, hashed)
```

- `argon2`/`bcrypt`; `md5`/`sha1` запрещены.
- Пароли никогда не логируются и не возвращаются в схемах (`Field(exclude=True)`).

## JWT

```python
from datetime import datetime, timedelta, timezone
from jose import JWTError, jwt

def create_access_token(subject: str, minutes: int = 15) -> str:
    now = datetime.now(timezone.utc)
    payload = {"sub": subject, "iat": now, "exp": now + timedelta(minutes=minutes)}
    return jwt.encode(payload, settings.secret_key.get_secret_value(), algorithm="HS256")
```

- Короткий срок жизни access-токена; refresh — отдельно.
- Алгоритм зафиксирован при decode (`algorithms=["HS256"]`) — иначе alg confusion.
- Секрет — из env (`SecretStr`), не хардкод.

```python
def decode_access_token(token: str) -> dict:
    try:
        return jwt.decode(token, settings.secret_key.get_secret_value(), algorithms=["HS256"])
    except JWTError as e:
        raise InvalidTokenError() from e
```

## OAuth2 / текущий пользователь

```python
from fastapi.security import OAuth2PasswordBearer
from fastapi import Depends

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/api/v1/auth/token")

async def get_current_user(token: str = Depends(oauth2_scheme)) -> UserOut:
    payload = decode_access_token(token)
    user = await user_service.get_by_id(payload["sub"])
    if user is None:
        raise InvalidTokenError()
    return user

CurrentUser = Annotated[UserOut, Depends(get_current_user)]
```

- Ошибки аутентификации — доменные (`InvalidTokenError` → 401).
- Права проверять на каждом роуте, не полагаться на скрытие URL.

## API key (внутренние эндпоинты)

```python
def require_api_key(x_api_key: str | None = Header(default=None, alias="X-API-Key")) -> None:
    if not x_api_key or not secrets.compare_digest(x_api_key, settings.api_key.get_secret_value()):
        raise InvalidApiKeyError()
```

Сравнение — `secrets.compare_digest` (защита от timing-атак).

## CORS

```python
app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.cors_origins,   # явный список
    allow_credentials=True,
    allow_methods=["GET", "POST"],
    allow_headers=["Authorization", "Content-Type"],
)
```

Не использовать `allow_origins=["*"]` с `allow_credentials=True`.

## OpenAPI/доки в проде

```python
app = FastAPI(
    docs_url=None if settings.is_prod else "/docs",
    redoc_url=None if settings.is_prod else "/redoc",
    openapi_url=None if settings.is_prod else "/openapi.json",
)
```

## Ввод и SQL

- Валидация входа — Pydantic (`extra="forbid"`, ограничения).
- SQL — только параметризованный/SQLAlchemy; f-строки в SQL запрещены.
- Ограничивать размер тела/загрузок; проверять тип файлов.
- Не отдавать внутренние ошибки и стектрейсы (см. `errors.md`).

## Секреты

- Только `pydantic-settings` из env; `os.getenv()` вне config запрещён.
- Не логировать токены/пароли/ключи/PII.
- Разные секреты для dev/staging/prod.

## Антипаттерны

| ❌ | ✅ |
|---|---|
| `hashlib.md5(password)` | `argon2`/`bcrypt` |
| Хардкод JWT-секрета | `SecretStr` из env |
| `jwt.decode` без `algorithms` | Явный список алгоритмов |
| `allow_origins=["*"]` + credentials | Явный список origin |
| Docs открыты в проде | `docs_url=None` |
| `==` для сравнения ключей | `secrets.compare_digest` |
| Пароль/токен в логах | Маскирование, `exclude=True` |
| Стектрейс в ответе | Общий handler + лог |

## Чек-лист

- [ ] Пароли хешируются современным алгоритмом.
- [ ] JWT: короткий срок, зафиксированный алгоритм, секрет из env.
- [ ] Права проверяются на каждом роуте.
- [ ] CORS ограничен; docs скрыты в проде.
- [ ] Вход валидируется; SQL параметризован.
- [ ] Секреты из env и не логируются.
- [ ] Общий handler не раскрывает детали.
