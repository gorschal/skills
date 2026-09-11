# Безопасность Django

Django-специфика OWASP Top 10. Расширенный чек-лист — в навыке `security`.

## Настройки (settings.py)

```python
DEBUG = False
ALLOWED_HOSTS = ["example.com", "www.example.com"]
SECRET_KEY = env("SECRET_KEY")          # только из окружения

SECURE_SSL_REDIRECT = True
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True
SECURE_HSTS_SECONDS = 31536000
SECURE_CONTENT_TYPE_NOSNIFF = True
X_FRAME_OPTIONS = "DENY"

AUTH_PASSWORD_VALIDATORS = [ ... ]      # не оставлять пустым
```

| ❌                              | ✅                    |
| ------------------------------- | --------------------- |
| `DEBUG = True` в проде          | `False`               |
| `ALLOWED_HOSTS = ["*"]`         | Только нужные домены  |
| Хардкод `SECRET_KEY`            | Из env                |
| `CORS_ALLOW_ALL_ORIGINS = True` | Явный список origin   |
| Пустой пароль БД                | Сильный пароль из env |

## Broken Access Control / IDOR

```python
# ❌ объект по id без проверки владельца
def document_detail(request, pk):
    return render(request, "doc.html", {"doc": Document.objects.get(pk=pk)})

# ✅ queryset ограничен пользователем
class DocumentDetailView(LoginRequiredMixin, DetailView):
    def get_queryset(self):
        return Document.objects.filter(owner=self.request.user)
```

- Все защищённые views — `@login_required`/`LoginRequiredMixin`.
- `get_queryset()` фильтрует по `request.user`; не отдавать чужие объекты.
- Не полагаться на скрытие URL — проверять права на объект.

## CSRF

- Не использовать `@csrf_exempt` без явного обоснования.
- В формах — `{% csrf_token %}`.
- AJAX — передавать CSRF-токен в заголовке.

## XSS

```python
# ❌
{{ user_bio|safe }}
{% autoescape off %}{{ content }}{% endautoescape %}
mark_safe(user_input)
```

- Не применять `|safe`/`mark_safe`/`autoescape off` к пользовательским данным.
- По умолчанию автоэкранирование включено — не отключать.

## SQL-инъекции

```python
# ❌ f-строки в SQL
cursor.execute(f"SELECT * FROM users WHERE name = '{name}'")
Model.objects.raw(f"SELECT * FROM {table}")
Model.objects.extra(where=[f"name = '{name}'"])

# ✅ ORM / параметры
User.objects.filter(name=name)
cursor.execute("SELECT * FROM users WHERE name = %s", [name])
```

Весь SQL — через ORM или параметризованные запросы.

## Mass assignment

```python
# ❌
class Meta:
    fields = "__all__"          # включая is_superuser/is_staff

User.objects.create(**request.POST)
```

- Явный список `fields` в формах/сериализаторах.
- Никогда не создавать модели из «сырого» `request.POST`/`request.data`.

## Загрузка файлов

- Проверять тип (по содержимому, не только расширение), размер и имя.
- Хранить файлы вне директорий кода; не использовать имя от пользователя как путь.
- Защита от path traversal: не подставлять ввод в `open(...)`.

## Аутентификация и секреты

- `AUTH_PASSWORD_VALIDATORS` настроен (длина, общность, сходство).
- Пароли — только через `set_password`/`create_user` (хеширование).
- Не логировать пароли, токены, номера карт.
- Небезопасная десериализация: `pickle.loads`, `yaml.load` без `SafeLoader` — запрещены.

## Чек-лист

- [ ] `DEBUG=False`, `ALLOWED_HOSTS` ограничен, `SECRET_KEY` из env.
- [ ] HTTPS enforced; secure cookies; HSTS; `X_FRAME_OPTIONS="DENY"`.
- [ ] CSRF-защита включена; `@csrf_exempt` обоснован.
- [ ] Нет `|safe`/`mark_safe` на пользовательских данных.
- [ ] SQL только через ORM/параметры.
- [ ] Права проверяются; `get_queryset()` фильтрует по пользователю.
- [ ] `fields` перечислены явно (нет mass assignment).
- [ ] Загрузки проверяются по типу/размеру/имени.
- [ ] Пароли валидируются; секреты не в логах.
- [ ] `SECRET_KEY`/ключи не в git.
