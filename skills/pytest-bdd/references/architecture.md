# Архитектура BDD-проекта

BDD — основное сквозное покрытие монорепозитория. Слои разделяют бизнес-язык,
связку и технику.

## Слои и ответственность

| Слой                   | Путь                      | Делает                         | Не делает            |
| ---------------------- | ------------------------- | ------------------------------ | -------------------- |
| Features               | `features/`               | Gherkin: бизнес-поведение      | технические детали   |
| Step Definitions       | `tests/step_definitions/` | Gherkin → Page Object / Client | селекторы, HTTP, SQL |
| Page Objects / Clients | `tests/clients/`          | UI/API-взаимодействие          | бизнес-ожидания      |
| Utils                  | `tests/utils/`            | генерация данных, конфиг       | шаги/ассерты         |
| Fixtures               | `conftest.py`             | ресурсы, контекст, cleanup     | бизнес-логика        |

Правило: **шаг вызывает один метод** Page Object / Client. Никакого
Selenium/httpx-кода в step-функциях.

## Структура проекта

```
project_root/
├── conftest.py                 # общие фикстуры (browser, api_client, context)
├── pytest.ini / pyproject.toml # маркеры, bdd_features_base_dir
├── features/
│   ├── auth/
│   ├── payments/
│   └── webhooks/
├── tests/
│   ├── clients/
│   │   ├── base_page.py
│   │   ├── base_api_client.py
│   │   ├── login_page.py
│   │   └── auth_client.py
│   ├── step_definitions/
│   │   ├── conftest.py
│   │   ├── common_steps.py
│   │   └── test_login_steps.py
│   └── utils/
│       ├── config.py
│       └── data_generators.py
├── data/                       # тестовые данные
├── reports/
└── screenshots/
```

Feature-файлы группируются по бизнес-домену; тестовые файлы со step definitions
могут иметь другую структуру — привязка через `scenarios()`/`@scenario`.

## Изоляция

- **Один сценарий = один независимый кейс.** Не зависит от порядка и других сценариев.
- Состояние — только через `target_fixture` или `context`.
- Очистка данных — `yield` + teardown в фикстурах.
- `scope="function"` по умолчанию.

## Теги и организация

- Теги Gherkin автоматически становятся pytest-маркерами (`@` снимается).
- С фильтрацией: `pytest -m "backend and login and successful"`.
- С `--strict-markers` все теги регистрируются в `markers` pytest-конфига.
- Имена тегов — python-совместимые (`@smoke`, не `@1st`).
- `@wip` — не гоняется в CI.

```ini
[pytest]
markers =
    smoke: критический путь
    regression: полная регрессия
    api: только API
    ui: только UI
    wip: в разработке (не в CI)
```

## Кастомные теги

```python
def pytest_bdd_apply_tag(tag, function):
    if tag == "todo":
        pytest.mark.skip(reason="Not implemented yet")(function)
        return True
    return None
```

## BDD как основное покрытие

- BDD проверяет всю бизнес-логику сквозным образом.
- Unit-тесты (навык `python-testing`) берут только логику вне BDD и критичные ветки.
- Не дублировать BDD-сценарии unit-тестами.

## Антипаттерны

| ❌                               | ✅                             |
| -------------------------------- | ------------------------------ |
| Технические детали в `.feature`  | Только бизнес-язык             |
| Вся логика в step-функции        | Page Object / Client           |
| Сценарии зависят от порядка      | Полная изоляция                |
| Глобальные переменные            | `context` / `target_fixture`   |
| Один feature на весь проект      | Разбиение по доменам           |
| Теги без регистрации в `markers` | `--strict-markers` + `markers` |

## Чек-лист

- [ ] Слои разделены: features / steps / clients / utils / fixtures.
- [ ] Шаги делегируют в Page Objects / Clients.
- [ ] Сценарии изолированы, `scope="function"`.
- [ ] Теги зарегистрированы как pytest-маркеры.
- [ ] `@wip` исключён в CI.
- [ ] BDD — основное покрытие, unit не дублирует.
