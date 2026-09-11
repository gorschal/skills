# Логирование, отчёты, CI

## Логирование

- Логировать значимые действия (клик, отправка формы, API-вызов).
- Использовать `logging` или `structlog`; не `print`.
- Логи — в `reports/`; при падении прикладывать к артефактам CI.

```python
import structlog
logger = structlog.get_logger(__name__)

logger.info("api_request", method="POST", path="/payments", status=201)
```

## Скриншот при падении

```python
from pathlib import Path

def pytest_bdd_step_error(request, feature, scenario, step, step_func, step_func_args, exception):
    browser = request.getfixturevalue("browser") if "browser" in request.fixturenames else None
    if browser:
        path = Path("screenshots") / f"{scenario.name}_{step.name}.png".replace(" ", "_")
        path.parent.mkdir(exist_ok=True)
        browser.save_screenshot(str(path))
```

- Обязательно для UI-тестов.
- Имя — из сценария и шага; папка создаётся.
- Артефакт сохраняется в CI.

## Теги и маркеры

```ini
[pytest]
addopts = --strict-markers
markers =
    smoke: критический путь
    regression: полная регрессия
    critical: блокирующий функционал
    api: только API
    ui: только UI
    wip: в разработке (не в CI)
```

- Теги Gherkin → pytest-маркеры; при `--strict-markers` регистрировать все.
- Запуск подмножеств: `pytest -m "smoke"`, `pytest -m "regression and not wip"`.

## Отчёты

```bash
pytest -v --gherkin-terminal-reporter
pytest --cucumberjson=reports/cucumber.json
pytest --html=reports/report.html --self-contained-html
```

- `--gherkin-terminal-reporter` — читаемый вывод шагов.
- `--cucumberjson` — Cucumber JSON для внешних систем.
- HTML-отчёт — для команды.

## Параллельный запуск

```bash
pytest -n auto --dist loadscope
```

- `loadscope` — сценарии одного scope в одном воркере (безопаснее для ресурсов).
- Параллельность требует полной изоляции сценариев.

## CI

```bash
pytest -m "not wip" -n auto --dist loadscope \
  --html=reports/report.html --self-contained-html \
  --cucumberjson=reports/cucumber.json
```

- Гонять только `not wip`.
- Артефакты: HTML/Cucumber-отчёт + `screenshots/` + логи.
- Падение любого сценария — красный CI.

## Антипаттерны

| ❌ | ✅ |
|---|---|
| `print()` в шагах | `logging`/`structlog` |
| Нет скриншота при падении UI | `pytest_bdd_step_error` |
| `@wip` в CI | `-m "not wip"` |
| Теги не зарегистрированы | `--strict-markers` + `markers` |
| Неизолированные сценарии в `-n auto` | полная изоляция |
| Отчёты не сохраняются | артефакты CI |

## Чек-лист

- [ ] Значимые действия логируются.
- [ ] Скриншот при падении UI-шага.
- [ ] Теги зарегистрированы, `@wip` исключён в CI.
- [ ] Отчёты (HTML/Cucumber) генерируются и сохраняются.
- [ ] Параллельный запуск безопасен (изоляция).
- [ ] Артефакты прикладываются к CI.
