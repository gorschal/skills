# Аутентификация и авторизация

## Пароли

```python
from passlib.context import CryptContext

pwd = CryptContext(schemes=["argon2", "bcrypt"], deprecated="auto")

hashed = pwd.hash(password)
ok = pwd.verify(password, hashed)
```

- `argon2`/`bcrypt`; `md5`/`sha1`/plaintext — запрещены.
- Валидация силы пароля; не хранить и не логировать открытый пароль.
- Сравнение — константное время (библиотека делает это сама).

## Rate limiting и lockout

- Ограничивать попытки входа (per IP/аккаунт): например, 5 попыток / 15 мин.
- Lockout/backoff на перебор; логировать срабатывания.
- Общее сообщение об ошибке: «Invalid credentials» — не раскрывать, существует ли
  пользователь.

## JWT

```python
from datetime import datetime, timedelta, timezone
from jose import JWTError, jwt

def create_access_token(subject: str, minutes: int = 15) -> str:
    now = datetime.now(timezone.utc)
    payload = {"sub": subject, "iat": now, "exp": now + timedelta(minutes=minutes)}
    return jwt.encode(payload, settings.secret_key.get_secret_value(), algorithm="HS256")

def decode(token: str) -> dict:
    try:
        return jwt.decode(
            token,
            settings.secret_key.get_secret_value(),
            algorithms=["HS256"],          # allowlist алгоритма
            audience="app", issuer="app",
        )
    except JWTError as e:
        raise InvalidTokenError() from e
```

- Короткий срок access-токена; refresh — отдельно, с ротацией.
- Алгоритм зафиксирован (защита от `alg=none`/confusion).
- Секрет — из env; не в токене и не в логах.
- Хранить токен в `HttpOnly`/`Secure` cookie, а не в JS-доступном месте.

## Сессии (cookie)

- `HttpOnly`, `Secure`, `SameSite=Lax/Strict`.
- Ротация session id при логине (защита от fixation).
- Таймаут неактивности; logout инвалидирует сессию.

## API keys

```python
import secrets

def require_api_key(provided: str) -> None:
    if not secrets.compare_digest(provided, settings.api_key.get_secret_value()):
        raise InvalidApiKeyError()
```

- Сравнение — `secrets.compare_digest` (защита от timing-атак).
- Ключи — с ограниченным scope и возможностью отзыва.

## Авторизация

```python
# объект доступен только владельцу
def get_document(self, doc_id: int, user_id: int) -> Document:
    doc = self.repo.get(doc_id, owner_id=user_id)   # фильтр по владельцу
    if doc is None:
        raise DocumentNotFoundError(doc_id)
    return doc
```

- Проверка на сервере для **каждого** объекта (нет IDOR).
- Роли/права — декларативно (`permission_classes`, `IsOwner`).
- Deny by default; не отдавать чужие объекты 404-м/403-м.

## Антипаттерны

| ❌                                                        | ✅                 |
| --------------------------------------------------------- | ------------------ |
| `md5(password)`                                           | `argon2`/`bcrypt`  |
| Разное сообщение для «нет пользователя»/«неверный пароль» | общее сообщение    |
| Нет rate limiting                                         | lockout/backoff    |
| `jwt.decode` без `algorithms`                             | явный allowlist    |
| Токен в localStorage                                      | `HttpOnly` cookie  |
| Проверка прав только на клиенте                           | серверная проверка |
| `==` для API-ключей                                       | `compare_digest`   |

## Чек-лист

- [ ] Пароли хешируются современным алгоритмом.
- [ ] Rate limiting/lockout на входе; общие ошибки.
- [ ] JWT: срок, алгоритм, аудитория, секрет из env.
- [ ] Cookie-флаги корректны; ротация сессии.
- [ ] API-ключи сравниваются безопасно.
- [ ] Авторизация на каждом объекте (нет IDOR).
