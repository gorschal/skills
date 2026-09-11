---
name: pytest-bdd
description: >
  Use when writing or reviewing BDD tests with pytest-bdd 8.x: Gherkin features,
  step definitions, target_fixture, parsers, Page Object Model, API clients,
  fixtures, tags, reporting and CI. Триггеры: pytest-bdd, BDD, Gherkin, feature,
  Scenario Outline, step definition, given/when/then, Page Object, Selenium,
  Playwright, parsers, target_fixture, cucumber. Для unit-тестов — навык
  python-testing.
license: MIT
compatibility: opencode
metadata:
  version: "1.0.0"
  domain: testing
  triggers: pytest-bdd, BDD, Gherkin, feature, scenario, step definition, Page Object, Selenium, Playwright, parsers, target_fixture
  role: specialist
  scope: implementation
  output-format: code
  related-skills: python, python-testing, fastapi, django, aiogram
---

# pytest-bdd

BDD на pytest-bdd 8.x: Gherkin — для бизнеса, шаги — тонкая связка, работа с
UI/API — в Page Objects / Clients. В монорепозитории **BDD — основное сквозное
покрытие** всей логики.

> Unit-тесты (пробелы BDD, критичные ветки) — навык `python-testing`.

## Когда применять

- Feature-файлы, step definitions, Page Objects, API clients.
- Фикстуры, теги, отчёты, параллельный запуск BDD.
- НЕ для unit-тестов и не для проверки реализации (BDD проверяет поведение).

## Ключевые принципы

1. **`.feature` — только бизнес-язык.** Никаких селекторов, URL, SQL.
2. **Step Definition — только вызов** Page Object / API Client.
3. **Локаторы, ожидания, HTTP — в Page Objects / Clients.**
4. **Один сценарий — один бизнес-кейс**, ≤ ~12–15 шагов.
5. **Полная изоляция** сценариев.
6. **`target_fixture`** — передача состояния между шагами (pytest-bdd 8.x).
7. **Теги → pytest markers**; в CI гоняем `not wip`.
8. **Никаких `time.sleep()`** — только явные ожидания.
9. BDD — основное покрытие; unit дополняет.

## Архитектура (слои)

| Слой                   | Путь                                          | Ответственность                        |
| ---------------------- | --------------------------------------------- | -------------------------------------- |
| Features               | `features/`                                   | Только Gherkin                         |
| Step Definitions       | `tests/step_definitions/`                     | Связка Gherkin → Page Object / Client  |
| Page Objects / Clients | `tests/clients/`                              | UI (Selenium/Playwright) и API (httpx) |
| Utils / Helpers        | `tests/utils/`                                | Генераторы данных, конфиг              |
| Fixtures               | `conftest.py`, `step_definitions/conftest.py` | Браузер, клиент, контекст              |

## Структура проекта

```
features/<domain>/*.feature
tests/
├── clients/            # Page Objects + API Clients
├── step_definitions/   # conftest.py, common_steps.py, test_*_steps.py
└── utils/
data/  reports/  conftest.py  pytest.ini
```

## Gherkin

```gherkin
@smoke @api
Feature: Создание платежа
  As a merchant
  I want to create a payment
  So that I can receive money

  Background:
    Given я авторизован как мерчант

  Scenario: Успешное создание платежа
    When я создаю платёж на сумму "100.00" в валюте "USD"
    Then статус платежа равен "pending"
    And я получаю payment_id
```

- Параметры — в двойных кавычках.
- `Background` — только `Given`, для общих предусловий.
- `Scenario Outline` + `Examples` — для вариаций; допустимы несколько таблиц с тегами.
- Один feature-файл — один `Feature`.
- Теги: `@smoke`, `@regression`, `@critical`, `@api`, `@ui`, `@wip`.

Подробно: [references/gherkin.md](references/gherkin.md).

## Step Definitions

```python
scenarios("../../features/auth/login.feature")

@given(parsers.parse("я на странице входа"), target_fixture="login_page")
def open_login_page(browser) -> LoginPage:
    return LoginPage(browser).open()

@then(parsers.parse('я вижу сообщение "{message}"'))
def check_message(login_page: LoginPage, message: str) -> None:
    assert login_page.get_error() == message
```

- Всегда `parsers.parse()` (или `parsers.re()`); type hints обязательны.
- `target_fixture="..."` — если шаг создаёт состояние для других шагов.
- Общие шаги — в `common_steps.py`/`conftest.py`.
- Один шаг — одно понятное действие; без Selenium/httpx внутри.

Подробно: [references/steps.md](references/steps.md).

## Page Objects / API Clients

```python
class BasePage:
    def __init__(self, driver, timeout: int = 10) -> None:
        self.wait = WebDriverWait(driver, timeout)

    def click(self, locator: tuple) -> None:
        self.wait.until(EC.element_to_be_clickable(locator)).click()
```

- Локаторы — константы класса.
- Никаких `time.sleep()`; только `WebDriverWait` + Expected Conditions.
- API-клиенты наследуют `BaseAPIClient`, логируют запросы/ответы.

Подробно: [references/page-objects.md](references/page-objects.md).

## Фикстуры и контекст

```python
@pytest.fixture
def context() -> dict:
    """Общий контекст сценария (токены, id)."""
    return {}

@pytest.fixture(scope="function")
def browser():
    driver = webdriver.Chrome(options=Options())
    yield driver
    driver.quit()
```

- Состояние между шагами — через `target_fixture` или `context`.
- `scope="function"` по умолчанию (изоляция).
- Очистка — `yield` + teardown.

Подробно: [references/fixtures-context.md](references/fixtures-context.md).

## Данные

- Тестовые данные — в `data/*.json` или генераторах.
- Прод-секреты запрещены; секреты тестовой среды — только в `data/`/`.env`.
- Вариации — `Scenario Outline`, не копипаста.

## Логирование, отчёты, CI

```bash
pytest -m "smoke"
pytest -m "regression and not wip"
pytest -n auto --dist loadscope
pytest -v --gherkin-terminal-reporter
pytest --cucumberjson=reports/cucumber.json
```

- При падении шага — обязательный скриншот (`pytest_bdd_step_error`).
- В CI: только `not wip`, артефакты — HTML/Cucumber-отчёт + скриншоты.

```python
def pytest_bdd_step_error(request, scenario, step, **kwargs):
    browser = request.getfixturevalue("browser")
    browser.save_screenshot(f"screenshots/{scenario.name}_{step.name}.png")
```

Подробно: [references/reporting-ci.md](references/reporting-ci.md).

## Запрещённые паттерны

| ❌ Запрещено                             | ✅ Правильно                          |
| ---------------------------------------- | ------------------------------------- |
| Логика/селекторы в `.feature`            | Только бизнес-язык                    |
| Selenium/httpx в step-функции            | Вызов Page Object / Client            |
| Шаг без `target_fixture`, но с возвратом | `target_fixture="..."`                |
| `time.sleep()`                           | `WebDriverWait` + Expected Conditions |
| Глобальные переменные                    | `context` / `target_fixture`          |
| Смешение UI + API + БД в одном шаге      | Разделение ответственности            |
| Дублирование шагов                       | `common_steps.py` / `conftest.py`     |
| Сценарий > 15 шагов                      | Разбить на несколько                  |
| `When`/`Then` в `Background`             | Только `Given`                        |
| `@wip` в CI                              | Гонять `not wip`                      |
| Прод-секреты в репозитории               | `data/`/`.env`                        |

## Чек-лист code review

- [ ] `.feature` содержит только бизнес-язык.
- [ ] Шаги используют `parsers.parse()`/`re()`; type hints есть.
- [ ] Шаги с возвратом используют `target_fixture`.
- [ ] Логика UI/API — в Page Object / Client.
- [ ] Нет `time.sleep()`; только явные ожидания.
- [ ] Сценарии изолированы; общие шаги вынесены.
- [ ] `Background` — только `Given`.
- [ ] Теги расставлены; `@wip` исключён в CI.
- [ ] Скриншот при падении шага.
- [ ] Нет хардкода прод-секретов.
- [ ] Docstrings у нетривиальных шагов/методов.
- [ ] Отчёт (HTML/Cucumber) и параллельный запуск настроены.

## Справочники

| Тема                               | Reference                                                        | Загружать когда                         |
| ---------------------------------- | ---------------------------------------------------------------- | --------------------------------------- |
| Архитектура, слои, структура, теги | [references/architecture.md](references/architecture.md)         | Организация проекта и слоёв             |
| Gherkin                            | [references/gherkin.md](references/gherkin.md)                   | Написание feature-файлов                |
| Step definitions                   | [references/steps.md](references/steps.md)                       | Шаги, парсеры, `target_fixture`, хуки   |
| Page Objects / Clients             | [references/page-objects.md](references/page-objects.md)         | UI/API-обёртки, ожидания                |
| Фикстуры и контекст                | [references/fixtures-context.md](references/fixtures-context.md) | Браузер, клиент, состояние              |
| Отчёты и CI                        | [references/reporting-ci.md](references/reporting-ci.md)         | Логи, скриншоты, отчёты, параллельность |

## Связанные навыки

- `python-testing` — unit-тесты для пробелов BDD.
- `python` — общие практики языка, docstrings, инструменты.
- `fastapi` / `django` / `aiogram` — тестируемые системы.
