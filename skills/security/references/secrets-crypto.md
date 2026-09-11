# Секреты и криптография

## Управление секретами

```python
from pydantic_settings import BaseSettings, SettingsConfigDict
from pydantic import SecretStr

class Settings(BaseSettings):
    secret_key: SecretStr
    database_url: str
    model_config = SettingsConfigDict(env_file=".env", case_sensitive=True)

settings = Settings()
```

- Секреты — только из окружения или secret manager (Vault/AWS Secrets Manager).
- `SecretStr` не отображается в `repr`/логах.
- `.env` — в `.gitignore`, права `600`, владелец — сервисный пользователь.
- Разные секреты для dev/staging/prod.
- При утечке — немедленная ротация.

## Сканирование секретов

```bash
gitleaks detect --source=.
trufflehog filesystem .
```

- Запускать в CI и pre-commit.
- Паттерны для поиска: `password=`, `api_key=`, `secret=`, `token=`,
  `AWS_SECRET`, `sk_live_`, `-----BEGIN ... PRIVATE KEY-----`.

## Криптография

- Только проверенные библиотеки (`cryptography`, `passlib`); без самодельных
  алгоритмов.
- Хеширование паролей — `argon2`/`bcrypt` (не шифрование).
- Шифрование данных — AES-GCM (аутентифицированное); ключи — вне кода.
- Случайность — `secrets`, не `random`.
- TLS 1.2+; сертификаты валидируются; HSTS.

```python
import secrets
token = secrets.token_urlsafe(32)
```

## Чувствительные данные

- Не логировать пароли, токены, номера карт, PII; маскировать.
- Не возвращать чувствительное в API (`Field(exclude=True)`).
- Шифровать at-rest то, что требует (PII, платёжные данные).
- Минимизировать сбор и хранение.

## Сканирование зависимостей

```bash
uv run pip-audit
uv run safety check
uv run bandit -r src
uv run ruff check --select S
trivy fs .
```

- Обновлять зависимости; фиксировать версии приложений.
- Критичные уязвимости блокируют merge.

## Антипаттерны

| ❌ | ✅ |
|---|---|
| Хардкод секрета | env/secret manager |
| Секрет в логах/ответе | маскирование/`SecretStr` |
| `.env` в git | `.gitignore`, права `600` |
| `random` для токенов | `secrets` |
| Самодельная крипто | проверенные библиотеки |
| `md5` для целостности/пароля | `argon2`/SHA-256+ |
| Устаревшие зависимости | регулярный `pip-audit` |
| Один секрет на все среды | раздельные секреты |

## Чек-лист

- [ ] Секреты вне кода, из env/secret manager.
- [ ] `.env` игнорируется, права ограничены.
- [ ] Секреты/PII не в логах и ответах.
- [ ] Крипто — проверенные библиотеки, современные алгоритмы.
- [ ] Токены — через `secrets`.
- [ ] Secret-scan и dependency-scan в CI.
- [ ] Ротация при утечке.
