# Статический анализ

Установка пакетов — **только с согласия пользователя**. Никогда не использовать
`pip install --break-system-packages`. Способы запуска: `uvx <tool>` (рекомендуется),
`pipx run <tool>`, или venv проекта.

## Качество кода (ruff)

```bash
uvx ruff check . --output-format=json > ruff.json
uvx ruff format --check .
```

`ruff` заменяет pylint/flake8/isort/pyupgrade. Настройка правил — в
`python/references/tooling.md`. Оценка: чем больше error/warning, тем ниже балл.

## Типизация (pyright)

```bash
uvx pyright
# опционально: uvx mypy . --ignore-missing-imports --no-error-summary
```

Зафиксировать число ошибок типизации и частые категории.

## Безопасность (bandit / ruff S)

```bash
uvx bandit -r . -f json --exclude "*/test*,*/venv/*,*/.venv/*" > bandit.json
# или: uvx ruff check --select S .
```

Разобрать `results`: HIGH / MEDIUM / LOW. Для каждой HIGH — `issue_text`,
`filename`, `line_number`.

## Сложность (radon)

```bash
uvx radon cc . -a -nb --exclude "venv,.venv"   # цикломатическая, только B и хуже
uvx radon mi . -nb --exclude "venv,.venv"      # maintainability
```

Отметить функции/модули ранга C и ниже — кандидаты на рефакторинг.

## Мёртвый код (vulture / ruff)

```bash
uvx vulture . --exclude "venv,.venv" --min-confidence 80
# или: uvx ruff check --select F401,F841 .
```

Неиспользуемые функции/переменные/импорты.

## Зависимости (pip-audit / safety)

```bash
uvx pip-audit
# или: uvx safety check -r requirements.txt
```

Если файла зависимостей нет — отметить, что проверка не выполнена.

## Покрытие тестами (опционально)

Покрытие требует запуска тестов — на статическом аудите опционально и только с
согласия. Иначе оценить косвенно (соотношение test-файлов к модулям, тесты
критичных путей). Подробнее — навык `python-testing`.

## Оценка

- **pylint-стиль (ruff):** score = 10 − (errors·2 + warnings·0.5 + conventions·0.1),
  не ниже 0.
- **bandit:** HIGH весомее; каждая HIGH — отдельная находка.
- **radon:** средняя сложность и maintainability — ориентир.
- **vulture:** неиспользуемое — кандидаты на удаление (проверить ложные срабатывания).

## Чек-лист

- [ ] Инструменты запущены без загрязнения системного Python.
- [ ] Упавшие инструменты отмечены в отчёте.
- [ ] Результаты суммированы (errors/warnings, HIGH/MEDIUM/LOW).
- [ ] Покрытие — только с согласия (или косвенная оценка).
