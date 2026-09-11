# Документирование (docstrings)

Docstrings — в коде, объясняют **«почему»**, а не пересказывают сигнатуру.
README и ADR — в навыке `documentation`.

## Матрица docstring по артефактам

| Артефакт                  | Детализация                                    | Язык  | Аудитория       | Обязательно                             |
| ------------------------- | ---------------------------------------------- | ----- | --------------- | --------------------------------------- |
| Unit-тесты                | Минимально, по существу                        | RU/EN | Разработчик     | Одна строка для нетривиального кейса    |
| Django views              | Максимально подробно: что, зачем, side effects | RU    | Мейнтейнер      | Поведение, вход/выход, побочные эффекты |
| FastAPI endpoints         | Как OpenAPI-описание (попадает в ReDoc)        | EN    | Потребитель API | Summary/description, коды, ошибки       |
| Сервисы                   | «Почему» + ограничения                         | RU    | Мейнтейнер      | Args/Returns/Raises/Side Effects        |
| Репозитории               | Сложные запросы и маппинги                     | RU    | Мейнтейнер      | Смысл запроса, особенности              |
| Модели/схемы              | Нетривиальные поля и валидаторы                | RU    | Мейнтейнер      | Инварианты, ограничения                 |
| `__init__`, геттеры, CRUD | Не документировать                             | —     | —               | —                                       |

## Сервисы — Google-style

```python
async def create_user(self, email: str, password: str) -> User:
    """Создаёт неактивного пользователя.

    Пользователь получает is_active=False до верификации email.

    Args:
        email: Адрес электронной почты (уникальный).
        password: Пароль в открытом виде (минимум 8 символов).

    Returns:
        User с is_active=False.

    Raises:
        UserAlreadyExistsError: Если email уже занят.

    Side Effects:
        Запись в БД, отправка welcome-email после коммита.
    """
```

- Документируем «почему» и ограничения, не дублируем сигнатуру словами.
- Ограничения/единицы/формат — даже если это «что»: они не выводятся из типа.

## Django views — подробно

```python
def signup_view(request: HttpRequest) -> HttpResponse:
    """Обрабатывает регистрацию пользователя.

    При GET показывает форму. При POST валидирует, создаёт неактивного
    пользователя через UserService и отправляет welcome-email после коммита.
    Бизнес-логика делегируется сервису, view отвечает только за HTTP.
    """
```

## FastAPI endpoints — English (OpenAPI/ReDoc)

Docstring эндпоинта становится `description` в OpenAPI. Пишем на английском, для
внешнего потребителя API; указываем коды/ошибки.

```python
@router.post("/users", response_model=UserOut, status_code=201)
async def create_user(payload: UserCreate, service: UserService = Depends(get_user_service)) -> UserOut:
    """Create a new inactive user.

    Raises:
        409: Email already registered.
        422: Invalid payload.
    """
```

- `summary` — из имени функции или `summary=`; `description` — docstring.
- Примеры/схемы — через `response_model`/`openapi_extra`, не в docstring.

## Unit-тесты — кратко

```python
async def test_rejects_expired_token() -> None:
    """Просроченный токен → Unauthorized."""
```

## Что не документировать

- `__init__`, `__str__`, `__repr__`.
- Одно-строчные геттеры/сеттеры.
- Простой CRUD без логики.
- Стандартные хуки фреймворка без доп. логики.

### Антипаттерн: дублирование сигнатуры

```python
# ❌ ничего не добавляет
def create_user(self, email: str) -> User:
    """Создаёт пользователя из email.

    Args:
        email: Email.

    Returns:
        User.
    """
```

## Принципы

- **«Почему», а не «что».** Код говорит, _что_; docstring объясняет _почему_.
- **Близко к коду.** Docstrings — в коде.
- **Аудитория определяет формат.** См. матрицу.
- **Устаревшее — помечать или удалять.**

## Чек-лист

- [ ] Детализация и язык docstring соответствуют матрице артефактов.
- [ ] FastAPI docstrings — английский, пригодны для ReDoc/OpenAPI.
- [ ] Django views — подробно, с побочными эффектами.
- [ ] Unit-тесты — кратко, без пересказа очевидного.
- [ ] Docstring не дублирует сигнатуру, объясняет «почему».

## См. также

- `documentation` — README, ADR.
