# Ввод/вывод и инъекции

## SQL-инъекции

```python
# ❌ f-строка/конкатенация
cursor.execute(f"SELECT * FROM users WHERE name = '{name}'")
Model.objects.raw(f"SELECT * FROM {table}")

# ✅ параметризация
cursor.execute("SELECT * FROM users WHERE name = %s", [name])

# ✅ ORM / SQLAlchemy Core
User.objects.filter(name=name)
await session.execute(select(User).where(User.name == name))
```

Весь SQL — через ORM/параметры. `extra()`/`raw()` с f-строками запрещены.

## XSS

```python
# ❌
{{ user_bio|safe }}
mark_safe(user_input)
{% autoescape off %}{{ content }}{% endautoescape %}

# ✅ автоэкранирование (по умолчанию)
{{ user_bio }}
```

- Не отключать автоэкранирование; не применять `safe`/`mark_safe` к вводу.
- Для HTML из пользователя — санитайзер с allowlist.
- CSP ограничивает источники скриптов.

## CSRF

- Cookie-сессии — с CSRF-токеном (`{% csrf_token %}`, `CSRF_COOKIE_SECURE`).
- Не использовать `@csrf_exempt` без явного обоснования.
- AJAX — токен в заголовке.

## Command injection

```python
# ❌
os.system(f"convert {filename}")
subprocess.run(f"ls {path}", shell=True)

# ✅ список аргументов, без shell
subprocess.run(["convert", filename], shell=False, check=True)
```

## Path traversal

```python
from pathlib import Path

# ❌
open(f"/uploads/{filename}")

# ✅ basename + проверка префикса
base = Path("/uploads").resolve()
target = (base / Path(filename).name).resolve()
if not target.is_relative_to(base):
    raise ValueError("invalid path")
```

## SSRF

```python
from urllib.parse import urlparse

# ❌
requests.get(user_url)

# ✅ allowlist схемы/хоста + блок внутренних адресов
parsed = urlparse(user_url)
if parsed.scheme not in {"https"} or parsed.hostname not in ALLOWED_HOSTS:
    raise ValueError("url not allowed")
```

Блокировать `localhost`, приватные диапазоны, `169.254.169.254` (метаданные).

## Десериализация

```python
# ❌
pickle.loads(user_input)
yaml.load(user_input)              # без SafeLoader
eval(user_input)

# ✅
json.loads(user_input)
yaml.safe_load(user_input)
Model.model_validate(user_input)   # Pydantic
```

## Загрузка файлов

- Проверять тип по содержимому (magic bytes), а не только расширение.
- Ограничивать размер; имя — генерировать, не доверять пользовательскому.
- Хранить вне директорий кода; не исполнять загруженное.

## Валидация ввода

```python
class UserCreate(BaseModel):
    email: EmailStr
    age: int = Field(ge=18, le=120)
    model_config = ConfigDict(extra="forbid")
```

- Валидировать на границе; `extra="forbid"` на командных схемах.
- Ограничивать длину/диапазоны; не доверять клиенту.

## Антипаттерны

| ❌ | ✅ |
|---|---|
| f-строка в SQL | параметризация/ORM |
| `|safe`/`mark_safe` | автоэкранирование |
| `shell=True` с вводом | список аргументов |
| `open(f"/uploads/{name}")` | `basename` + проверка |
| `requests.get(user_url)` | allowlist + блок внутренних |
| `pickle`/`eval`/`yaml.load` | JSON/Pydantic/`safe_load` |
| Доверие расширению файла | проверка содержимого |

## Чек-лист

- [ ] SQL параметризован/через ORM.
- [ ] XSS: автоэкранирование, без `safe` на вводе.
- [ ] CSRF-токены; `@csrf_exempt` обоснован.
- [ ] Команды — без shell с вводом.
- [ ] Path traversal и SSRF закрыты.
- [ ] Десериализация безопасна.
- [ ] Загрузки проверяются по содержимому/размеру/имени.
