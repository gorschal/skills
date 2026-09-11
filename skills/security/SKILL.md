---
name: security
description: >
  Use when implementing or reviewing security in Python services: OWASP Top 10,
  authentication/authorization, password hashing, JWT, input validation, SQL
  injection, XSS/CSRF, IDOR, SSRF, secrets management, dependency scanning and
  security review. Триггеры: security, OWASP, vulnerability, injection, XSS,
  CSRF, IDOR, SSRF, JWT, password hashing, secrets, bandit, semgrep, gitleaks,
  security audit. Фреймворк-специфика — навыки django/fastapi.
license: MIT
compatibility: opencode
metadata:
  version: "1.0.0"
  domain: security
  triggers: security, OWASP, injection, XSS, CSRF, IDOR, SSRF, JWT, secrets, SAST, audit
  role: specialist
  scope: implementation
  output-format: code
  related-skills: python, django, fastapi, faststream, aiogram
---

# Security

Безопасность Python-сервисов: OWASP Top 10, аутентификация, валидация ввода,
секреты, сканирование и security review. Применяется ко всем фреймворкам стека.

> Фреймворк-специфика настроек — в навыках `django` (`references/security.md`) и
> `fastapi` (`references/security.md`).

## Когда применять

- Реализация аутентификации/авторизации, хеширования, JWT.
- Ревью кода на уязвимости (injection, XSS, IDOR, SSRF).
- Управление секретами, сканирование зависимостей и секретов.
- Security review / аудит с отчётом.

## Ключевые принципы

1. **Не доверять вводу.** Валидировать на границе (Pydantic/формы).
2. **Deny by default.** Права проверяются на сервере для каждого объекта.
3. **Defense in depth.** Несколько независимых барьеров.
4. **Наименьшие привилегии.** У пользователей/сервисов/БД — минимум прав.
5. **Секреты — вне кода.** Только окружение/secret manager.
6. **Не отдавать внутренности.** Стектрейсы и детали — только в логи.
7. **Логировать события безопасности** (неудачный вход, эскалация).
8. **Проверять зависимости** регулярно (SAST/SCA).

## OWASP Top 10 (Python)

| # | Уязвимость | Профилактика |
|---|---|---|
| A01 | Broken Access Control | проверка владельца/роли, deny by default |
| A02 | Cryptographic Failures | современные алгоритмы, TLS, секреты из env |
| A03 | Injection | параметризованный SQL, без `eval`/shell |
| A04 | Insecure Design | threat model, ограничения, rate limiting |
| A05 | Security Misconfiguration | `DEBUG=False`, headers, CORS |
| A06 | Vulnerable Components | `pip-audit`/`safety`, обновления |
| A07 | Auth Failures | хеширование, lockout, короткие сессии |
| A08 | Data Integrity | JSON/Pydantic, без `pickle` |
| A09 | Logging Failures | логировать security-события |
| A10 | SSRF | allowlist URL, запрет внутренних адресов |

Подробно: [references/owasp.md](references/owasp.md).

## Аутентификация и авторизация

```python
from passlib.context import CryptContext

pwd = CryptContext(schemes=["argon2", "bcrypt"], deprecated="auto")
hashed = pwd.hash(password)
ok = pwd.verify(password, hashed)
```

- Пароли — `argon2`/`bcrypt`; `md5`/`sha1`/открытый текст запрещены.
- Rate limiting и lockout на auth-эндпоинтах.
- JWT: короткий срок, зафиксированный алгоритм (`algorithms=["HS256"]`),
  секрет из env; `aud`/`iss`.
- Сессии: `HttpOnly`, `Secure`, `SameSite`; защита от fixation.
- Сообщение об ошибке входа — общее (не раскрывать существование пользователя).
- Авторизация — на каждом объекте (`get_queryset` по пользователю, проверка владельца).

Подробно: [references/auth.md](references/auth.md).

## Ввод/вывод и инъекции

```python
# ❌ SQL-инъекция
cursor.execute(f"SELECT * FROM users WHERE name = '{name}'")

# ✅ параметризация / ORM
cursor.execute("SELECT * FROM users WHERE name = %s", [name])
User.objects.filter(name=name)
```

- SQL — только параметризованно; f-строки в SQL запрещены.
- Команды — списком аргументов, без `shell=True` с вводом.
- XSS: не использовать `|safe`/`mark_safe`/`autoescape off` на вводе.
- CSRF: токены для cookie-сессий; не отключать `@csrf_exempt` без причины.
- Path traversal: `Path.basename` + проверка префикса.
- SSRF: allowlist доменов/схем; блокировать внутренние адреса.
- Десериализация: только JSON/Pydantic; `pickle`/`yaml.load`/`eval` запрещены.
- Загрузки: проверять тип (по содержимому), размер, имя.

Подробно: [references/input-output.md](references/input-output.md).

## Секреты и криптография

- Секреты — только из окружения (`pydantic-settings`/`SecretStr`) или secret manager.
- `.env` в `.gitignore`, права `600`; при утечке — немедленная ротация.
- Не логировать пароли/токены/PII; маскировать.
- Крипто — проверенные библиотеки; никаких собственных алгоритмов.
- TLS везде; HSTS; secure cookies.
- Сканирование секретов (`gitleaks`/`trufflehog`) в CI.

Подробно: [references/secrets-crypto.md](references/secrets-crypto.md).

## Зависимости и SAST

```bash
uv run bandit -r src            # статический анализ
uv run pip-audit                # уязвимые зависимости
uv run ruff check --select S    # flake8-bandit
semgrep --config=auto .         # мультиязычный SAST
gitleaks detect --source=.      # утечки секретов
```

- Запускать в CI; критичные находки блокируют merge.
- Обновлять зависимости; фиксировать версии приложений.

## Логирование безопасности

- Логировать: неудачные входы, отказы в доступе, эскалацию привилегий, изменения
  прав, срабатывание rate limit.
- **Не** логировать секреты/PII; маскировать.
- Алерты на критические события.

## Security review

1. **Scope** — карта поверхности атаки, критические пути; письменная авторизация
   для активного тестирования.
2. **Scan** — SAST, зависимости, секреты.
3. **Manual review** — auth, ввод, крипто (инструменты не видят контекст).
4. **Classify** — severity (Critical/High/Medium/Low) по CVSS; PoC в пределах scope.
5. **Report** — файл/строка, impact, remediation; критические — сразу.

Формат находки: `ID`, severity (CVSS), title, file:line, description, impact,
remediation, references (CWE/OWASP).

Подробно: [references/review.md](references/review.md).

## Запрещённые паттерны

| ❌ Запрещено | ✅ Правильно |
|---|---|
| `md5`/`sha1`/plaintext для паролей | `argon2`/`bcrypt` |
| f-строка в SQL | параметризация/ORM |
| `eval`/`exec`/`pickle` на вводе | JSON/Pydantic |
| `shell=True` с вводом | список аргументов |
| `|safe`/`mark_safe` на вводе | автоэкранирование |
| Хардкод секретов | env/secret manager |
| Секреты/PII в логах | маскирование |
| Стектрейс в ответе | общий handler |
| Нет проверки владельца (IDOR) | авторизация на объекте |
| `DEBUG=True`/`ALLOWED_HOSTS=*` в проде | продовые настройки |
| `requests.get(user_url)` без проверки (SSRF) | allowlist |
| `@csrf_exempt` без причины | CSRF-защита |

## Чек-лист

- [ ] Пароли хешируются современным алгоритмом.
- [ ] Auth-эндпоинты защищены rate limiting/lockout.
- [ ] JWT/сессии: срок, алгоритм, флаги cookie.
- [ ] Авторизация проверяется на каждом объекте (нет IDOR).
- [ ] SQL параметризован; нет `eval`/shell/`pickle`.
- [ ] XSS/CSRF закрыты; нет `safe` на вводе.
- [ ] SSRF/path traversal/загрузки проверяются.
- [ ] Секреты из env, не в коде/логах/git.
- [ ] SAST/SCA/secret-scan в CI.
- [ ] Логируются security-события, без PII.
- [ ] Продовые настройки безопасности (DEBUG, headers, CORS).

## Справочники

| Тема | Reference | Загружать когда |
|---|---|---|
| OWASP Top 10 | [references/owasp.md](references/owasp.md) | Профилактика уязвимостей |
| Аутентификация/авторизация | [references/auth.md](references/auth.md) | Пароли, JWT, сессии, права |
| Ввод/вывод и инъекции | [references/input-output.md](references/input-output.md) | SQL, XSS, SSRF, десериализация |
| Секреты и крипто | [references/secrets-crypto.md](references/secrets-crypto.md) | Секреты, TLS, сканирование |
| Security review | [references/review.md](references/review.md) | Аудит, severity, отчёт |

## Связанные навыки

- `python` — общие практики; `references/security.md`.
- `django` / `fastapi` — фреймворк-специфика безопасности.
- `python-audit` — аудит готовности проекта.
