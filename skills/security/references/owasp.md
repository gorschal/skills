# OWASP Top 10 (Python)

Профилактика уязвимостей с примерами на Python-стеке.

## A01: Broken Access Control

```python
# ❌ объект по id без проверки владельца
user = User.objects.get(id=user_id)

# ✅ фильтр по текущему пользователю
user = User.objects.get(id=user_id, owner=request.user)
```

- Deny by default; проверять права на сервере для каждого объекта.
- `get_queryset()` фильтрует по пользователю (Django/FastAPI).
- Не полагаться на скрытие URL/ID.

## A02: Cryptographic Failures

```python
# ❌
hashlib.md5(password)
SECRET_KEY = "hardcoded"

# ✅
pwd = CryptContext(schemes=["argon2"], deprecated="auto")
```

- Пароли — `argon2`/`bcrypt`; TLS везде; секреты из env.
- Современные алгоритмы (AES-GCM, RSA ≥ 2048); без самодельной крипто.

## A03: Injection

```python
# ❌ SQL
cursor.execute(f"SELECT * FROM users WHERE name = '{name}'")

# ✅ параметризация
cursor.execute("SELECT * FROM users WHERE name = %s", [name])
```

- SQL — параметризованно/ORM.
- Команды — списком аргументов (`subprocess.run([...], shell=False)`).
- Без `eval`/`exec`; шаблоны — без пользовательского ввода как шаблона.

## A04: Insecure Design

- Threat model до реализации.
- Rate limiting, квоты, лимиты размера.
- Токены сброса пароля — случайные (`secrets`), с TTL и одноразовые.

## A05: Security Misconfiguration

```python
DEBUG = False
ALLOWED_HOSTS = ["example.com"]
SECURE_SSL_REDIRECT = True
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True
SECURE_HSTS_SECONDS = 31536000
X_FRAME_OPTIONS = "DENY"
```

- Продовые настройки обязательны; дефолты отключены.
- CORS — явный allowlist; docs/админка — не наружу.

## A06: Vulnerable Components

```bash
uv run pip-audit
uv run bandit -r src
```

- Регулярно сканировать и обновлять зависимости.

## A07: Identification & Auth Failures

- Хеширование паролей, lockout/rate limit, MFA где нужно.
- Общие сообщения об ошибке входа (не раскрывать существование пользователя).
- Короткие сессии; защита от fixation.

## A08: Software & Data Integrity

```python
# ❌
pickle.loads(user_input)
yaml.load(user_input)          # без SafeLoader

# ✅
json.loads(user_input)
yaml.safe_load(user_input)
```

- Только JSON/Pydantic для недоверенных данных.
- Подписывать артефакты/обновления.

## A09: Logging & Monitoring Failures

- Логировать security-события (неудачный вход, отказ, эскалация).
- Не логировать секреты/PII.
- Алерты на критические события.

## A10: SSRF

```python
# ❌
requests.get(user_url)

# ✅ allowlist домена/схемы + блок внутренних адресов
if urlparse(url).hostname not in ALLOWED_HOSTS:
    raise ValueError("host not allowed")
```

- Allowlist доменов/схем; блокировать `localhost`, приватные диапазоны, метаданные
  облака (169.254.169.254).

## Антипаттерны

| ❌                          | ✅                     |
| --------------------------- | ---------------------- |
| Проверка прав на клиенте    | серверная проверка     |
| `md5`/plaintext             | `argon2`/`bcrypt`      |
| Конкатенация SQL            | параметризация         |
| `pickle`/`eval`             | JSON/Pydantic          |
| Дефолтные настройки в проде | хардненинг             |
| Устаревшие зависимости      | регулярный `pip-audit` |
| Нет логов безопасности      | логирование событий    |
| `requests.get(user_url)`    | allowlist              |

## Чек-лист

- [ ] Авторизация на каждом объекте (нет IDOR).
- [ ] Современная криптография; TLS.
- [ ] SQL/команды/шаблоны без инъекций.
- [ ] Rate limiting и лимиты.
- [ ] Продовые настройки безопасности.
- [ ] Зависимости сканируются.
- [ ] Auth устойчив (lockout, общие ошибки).
- [ ] Десериализация безопасна.
- [ ] Логи безопасности без PII.
- [ ] SSRF-защита.
