# Step Definitions (pytest-bdd 8.x)

Шаг — тонкая связка Gherkin и Page Object / Client.

## Декораторы

```python
from pytest_bdd import given, when, then, parsers, scenarios

scenarios("features/auth/login.feature")   # привязка всех сценариев файла

@given("я авторизован", target_fixture="auth")
def auth(api_client) -> dict:
    return api_client.login("user@example.com", "secret")

@when(parsers.parse('я открываю "{path}"'))
def open_path(path: str, login_page: LoginPage) -> None:
    login_page.open(path)

@then(parsers.parse('я вижу "{text}"'))
def see_text(text: str, login_page: LoginPage) -> None:
    assert login_page.get_text() == text
```

## Парсеры

| Парсер | Назначение | Пример |
|---|---|---|
| `string` (по умолчанию) | точное совпадение | `@given("я на странице")` |
| `parsers.parse` | именованные поля `{x:Type}` | `parsers.parse("сумма {amount:d}")` |
| `parsers.cfparse` | кардинальность `+ * ?` | `cfparse("{tags:Tag+}")` |
| `parsers.re` | regex с группами `(?P<name>...)` | `parsers.re(r"(?P<n>\d+)")` |

```python
@given(
    parsers.cfparse("есть {start:Number} огурцов", extra_types={"Number": int}),
    target_fixture="cucumbers",
)
def cucumbers(start: int) -> dict:
    return {"start": start, "eat": 0}

@then(parsers.re(r"осталось (?P<left>\d+)"), converters={"left": int})
def left(left: int, cucumbers: dict) -> None:
    assert cucumbers["start"] - cucumbers["eat"] == left
```

- Использовать `parsers.parse`/`re` вместо `string` при параметрах.
- Конвертеры — `extra_types` (cfparse) / `converters` (re).
- Имена `datatable` и `docstring` зарезервированы.

## target_fixture

Шаг, создающий состояние, обязан объявить `target_fixture` — иначе возврат
не станет фикстурой (в 8.x step-аргументы больше не фикстуры).

```python
@given("есть статья", target_fixture="article")
def article() -> Article:
    return Article()

@when("я публикую статью")
def publish(article: Article) -> None:
    article.publish()
```

`when`/`then` тоже могут отдавать фикстуру (например, результат HTTP-запроса):

```python
@when("я удаляю статью", target_fixture="response")
def delete(article: Article, http_client) -> httpx.Response:
    return http_client.delete(f"/articles/{article.uid}")

@then("запрос успешен")
def success(response: httpx.Response) -> None:
    assert response.status_code == 200
```

## Привязка сценариев

```python
# автоматически все сценарии из папки/файла
scenarios("features", "other/some.feature")

# вручную — один сценарий
from pytest_bdd import scenario

@scenario("features/login.feature", "Успешный вход")
def test_login() -> None:
    pass
```

Ручную привязку писать **до** `scenarios(...)`, иначе перезапишется.

## Переиспользование

- Общие шаги — в `conftest.py`/`common_steps.py`.
- Шаг может переиспользовать фикстуру под другим именем.
- Алиасы: несколько декораторов на одну функцию.

```python
@given("у меня есть статья")
@given("есть статья")
def article(author, target_fixture="article"):
    return create_article(author)
```

## Хуки

| Хук | Когда |
|---|---|
| `pytest_bdd_before_scenario` | перед сценарием |
| `pytest_bdd_after_scenario` | после (даже при падении) |
| `pytest_bdd_before_step` | перед шагом |
| `pytest_bdd_before_step_call` | перед вызовом с аргументами |
| `pytest_bdd_after_step` | после успешного шага |
| `pytest_bdd_step_error` | шаг упал (скриншот, диагностика) |
| `pytest_bdd_step_func_lookup_error` | шаг не найден |
| `pytest_bdd_apply_tag` | кастомная обработка тега |

## Конфигурация путей

```ini
[pytest]
bdd_features_base_dir = features/
```

Или per-scenario: `@scenario("foo.feature", "...", features_base_dir="./local/")`.

## Генерация кода

```bash
pytest-bdd generate features/some.feature > tests/functional/test_some.py
pytest --generate-missing --feature features tests/functional
```

## Миграция (важно для 8.x)

- Примеры на уровне feature и вертикальные таблицы — удалены.
- `<param>` в шагах → `parsers.parse("{param}")`.
- Step-аргументы больше не фикстуры → `target_fixture`.
- `strict_gherkin` удалён.
- Нельзя совмещать `Scenario Outline` и pytest-параметризацию.

## Антипаттерны

| ❌ | ✅ |
|---|---|
| `string`-парсер там, где есть параметры | `parsers.parse`/`re` |
| Возврат без `target_fixture` | `target_fixture="..."` |
| Selenium/httpx в шаге | Page Object / Client |
| Шаг делает несколько действий | Один action |
| Общие шаги в каждом файле | `conftest.py`/`common_steps.py` |
| `@given("there are <n>")` (v4-стиль) | `parsers.parse("there are {n}")` |

## Чек-лист

- [ ] Парсеры используются для всех параметров.
- [ ] Шаги с возвратом объявляют `target_fixture`.
- [ ] Type hints на всех шагах.
- [ ] Общие шаги вынесены в `conftest.py`.
- [ ] Нет технического кода в шагах.
- [ ] Соблюдён синтаксис 8.x (без `<param>` и step-as-fixture).
