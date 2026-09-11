# Инструменты

Стандартный набор: `uv` (пакеты и запуск), `ruff` (линт + формат),
`pyright` (типы), `pytest` (тесты). Конфиг — в `pyproject.toml`.

## uv

```bash
uv init
uv add httpx pydantic
uv add --dev pytest pytest-asyncio ruff pyright
uv sync
uv run pytest -v
uv run ruff check .
```

- Все команды — через `uv run` (единый lock и окружение).
- В Docker/CI: `docker compose exec app uv run ...`.

## ruff

Заменяет flake8, isort, pyupgrade, autoflake, pydocstyle и форматтер.

```toml
[tool.ruff]
line-length = 100
target-version = "py312"
exclude = [".venv", "dist", "build"]

[tool.ruff.lint]
select = [
  "E", "W",   # pycodestyle
  "F",        # pyflakes
  "I",        # isort
  "B",        # bugbear
  "C4",       # comprehensions
  "UP",       # pyupgrade
  "SIM",      # simplify
  "N",        # pep8-naming
  "S",        # bandit (security)
  "DTZ",      # datetimez
  "T20",      # print
  "RUF",      # ruff-specific
]
ignore = ["E501"]           # длину строк держит форматтер

[tool.ruff.lint.per-file-ignores]
"tests/**/*.py" = ["S101"]  # assert в тестах
"__init__.py" = ["F401"]    # re-export

[tool.ruff.lint.isort]
known-first-party = ["app"]

[tool.ruff.lint.pydocstyle]
convention = "google"
```

```bash
uv run ruff check .            # линт
uv run ruff check . --fix      # безопасные автофиксы
uv run ruff format .           # формат (вместо black)
uv run ruff format --check .   # проверка формата (CI)
uv run ruff rule S101          # объяснение правила
```

### Fix safety

- `--fix` применяет только **safe** фиксы (семантика сохраняется).
- `--unsafe-fixes` — только осознанно, после ревью диффа.
- `fixable`/`unfixable` — тонкая настройка.

### Подавление

```python
x = 1  # noqa: F841          # конкретное правило
# ruff: noqa: F401           # на весь файл
```

- `# noqa` без кода — запрещено (скрывает всё).
- Неиспользуемые `noqa` ловятся `RUF100`.

## pyright

```toml
[tool.pyright]
pythonVersion = "3.12"
typeCheckingMode = "strict"
```

```bash
uv run pyright
```

Любая ошибка типов должна быть устранена до сдачи.

## pre-commit

```yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.8.0
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format
  - repo: https://github.com/RobertCraigie/pyright-python
    rev: v1.1.390
    hooks:
      - id: pyright
```

```bash
uv run pre-commit install
uv run pre-commit run --all-files
```

## CI (пример)

```yaml
- run: uv sync --frozen
- run: uv run ruff check .
- run: uv run ruff format --check .
- run: uv run pyright
- run: uv run pytest --cov --cov-report=term-missing
```

## Единый прогон перед сдачей

```bash
uv run ruff check . && uv run ruff format --check . && uv run pyright && uv run pytest
```

## Чек-лист

- [ ] Зависимости через `uv`, lock зафиксирован.
- [ ] `ruff check` и `ruff format --check` проходят.
- [ ] `pyright` (strict) проходит.
- [ ] `pytest` зелёный.
- [ ] Pre-commit/CI настроены.
- [ ] `# noqa` — только с кодом правила и причиной.
