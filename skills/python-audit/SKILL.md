---
name: python-audit
description: >
  Use to audit a Python project's readiness before deploy: incomplete code and
  stubs, critical problems, dead code, weak spots. Static analysis (ruff,
  pyright, bandit, radon, vulture, pip-audit) plus a manual 8-category review,
  with a scored report and verdict. Триггеры: аудит проекта, готовность к
  деплою, MVP готовность, найти незавершённый код, критические проблемы,
  оценить законченность, production-ready. Уровень проекта, не диффа.
  Django-специфика — линза `django-lens`; безопасность — навык `security`.
license: MIT
compatibility: opencode
metadata:
  version: "1.0.0"
  domain: quality
  triggers: project audit, readiness, production-ready, incomplete code, critical issues, scoring, MVP
  role: auditor
  scope: review
  output-format: report
  related-skills: security, python-testing, django, postgres, migration-safety, documentation
---

# Python Audit

Проверка заявленной готовности против фактической: проект выглядит законченным —
разобраться, так ли это. Pipeline: статический анализ → ручной review → отчёт с
баллами и вердиктом.

Это **статический аудит**: код только читается и анализируется инструментами.
**Не запускай** тесты, `manage.py` или приложение.

## Когда применять

- «Проверь готовность к деплою», «отчёт о готовности MVP».
- «Найти незавершённый код и критические проблемы», оценить законченность.
- Уровень проекта, не диффа (одно изменение — code review).

## Шаг 0. Разведка

Используй Grep/Glob/Read (не POSIX-утилиты):

- **Структура**: `Glob` по `**/*.py` — размер.
- **Фреймворк**: `Grep` по `fastapi|django|flask`.
- **Зависимости**: `Read` `pyproject.toml`/`requirements.txt`.
- **Инфраструктура**: `Glob` `Dockerfile`, `docker-compose.yml`, `.github/workflows/*`.
- **Тесты**: `Glob` `**/test_*.py`, `**/conftest.py`.
- **Конфиги**: `Glob` `.env*`, `*.ini`, `*.cfg`.

Прочитай ключевые файлы: точка входа (`main.py`/`manage.py`), конфиг, роутинг, модели.

## Шаг 1. Статический анализ

Инструменты (установка — только с согласия; предпочитать `uvx`/`pipx`/venv, никогда
`pip install --break-system-packages`):

```bash
uvx ruff check .                 # качество/линт (заменяет pylint/flake8)
uvx ruff format --check .        # формат
uvx pyright                      # типы (основной); mypy — опционально
uvx bandit -r src                # безопасность (или ruff --select S)
uvx radon cc . -a -nb            # цикломатическая сложность
uvx vulture . --min-confidence 80 # мёртвый код
uvx pip-audit                    # уязвимые зависимости
```

Если инструмент не установился/упал — отметь в отчёте и двигайся дальше.

Подробно: [references/static-analysis.md](references/static-analysis.md).

## Шаг 2. Ручной review — 8 испытаний

Оцени каждую категорию 0–10. Для находки: файл, строка, проблема, решение.

| # | Испытание | Что смотрим |
|---|---|---|
| 1 | Архитектура | слои, DI, конфиг из env, циклические импорты |
| 2 | Обработка ошибок | доменные исключения, global handler, нет `except:` |
| 3 | Безопасность | валидация, SQLi, auth/authz, CORS, секреты, rate limit |
| 4 | Производительность | async I/O, пулы, индексы, N+1, кэш, пагинация |
| 5 | Тестирование | покрытие, unit/BDD, edge-cases, моки |
| 6 | Инфраструктура | Docker, CI, миграции, health, graceful shutdown |
| 7 | Качество кода | стиль, типы, docstrings, размеры, константы |
| 8 | Документация/DX | README, OpenAPI, `.env.example`, «почему» |

Полный чек-лист с антипаттернами — [references/manual-review.md](references/manual-review.md).

## Шаг 3. Подсчёт баллов

| Испытание | Вес | Баллы (0-10) | Взвешенный |
|---|---|---|---|
| Архитектура | 15% | ? | ? |
| Обработка ошибок | 10% | ? | ? |
| Безопасность | 20% | ? | ? |
| Производительность | 15% | ? | ? |
| Тестирование | 15% | ? | ? |
| Инфраструктура | 10% | ? | ? |
| Качество кода | 10% | ? | ? |
| Документация | 5% | ? | ? |
| **ИТОГО** | **100%** | | **?/10** |

Градации: **9-10** production-ready · **7-8** хороший, точечные улучшения ·
**5-6** MVP, серьёзная доработка · **3-4** прототип · **0-2** рефакторинг с нуля.

## Шаг 4. Отчёт

Сохрани в `docs/audit-YYYY-MM-DD.md`. Шаблон — [references/report.md](references/report.md).

## Шаг 5. Презентация

1. Показать итоговый балл и таблицу в чате.
2. Топ-3 критичных проблемы с решениями.
3. Предложить исправить критичное сейчас.

## Адаптация под фреймворки

- **FastAPI:** `Depends`, Pydantic на всех эндпоинтах, нет blocking I/O в `async`,
  `lifespan` вместо `on_event`, BackgroundTasks/очередь.
- **Django:** нет логики во views, `select_related`/`prefetch_related`, settings
  по окружениям, `ALLOWED_HOSTS`/`SECURE_*`, миграции без конфликтов.
- **FastStream/боты:** идемпотентность, ack/nack, graceful shutdown, один инстанс.

Полный Django-разбор — линза `django-lens` ниже.

## Django-линза

Глубокий разбор Django-проекта по линзам: `architecture`, `security`, `cleanup`,
`legacy`, `deploy`, `tests`, `tasks` (Django Tasks, **не** Celery). Каждая линза —
набор проверок; грузить только нужную.

Подробно: [references/django-lens.md](references/django-lens.md).

## Принципы

- **Конкретика:** «файл X, строка Y, проблема, решение» — не «улучшите безопасность».
- **Приоритизация:** критичное → важное → nice-to-have.
- **Контекст:** стадия проекта (MVP vs зрелый прод).
- **Баланс:** отмечать и сильные стороны.
- **Actionable:** каждая находка — конкретное действие.
- **Без эмодзи** в отчёте.
- **Не править сам:** аудитор находит и рекомендует; правки — по отдельной просьбе.

## Границы

- Только аудит и отчёт; исправления — по отдельной задаче.
- Уровень проекта, не диффа.
- Глубокий security-аудит — навык `security`; тесты — `python-testing`/`pytest-bdd`.

## Справочники

| Тема | Reference | Загружать когда |
|---|---|---|
| Статанализ | [references/static-analysis.md](references/static-analysis.md) | Команды инструментов, разбор вывода |
| Ручной review (8 испытаний) | [references/manual-review.md](references/manual-review.md) | Оценка категорий, антипаттерны |
| Отчёт и баллы | [references/report.md](references/report.md) | Шаблон отчёта, градации |
| Django-линза | [references/django-lens.md](references/django-lens.md) | Глубокий разбор Django |

## Связанные навыки

- `security` — расширенный security-аудит.
- `python-testing` / `pytest-bdd` — оценка тестов.
- `django` / `fastapi` — конвенции, на которые сверяемся.
- `postgres`, `migration-safety`, `documentation` — отдельные линзы.
