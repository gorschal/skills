# skills

[![opencode](https://img.shields.io/badge/-opencode-000000?style=flat)](https://opencode.ai)
[![Python](https://img.shields.io/badge/-Python_3.12-3776AB?style=flat&logo=python&logoColor=white)](https://www.python.org/)
[![License](https://img.shields.io/badge/-MIT-green?style=flat)](https://opensource.org/licenses/MIT)

Набор навыков (agent skills) для **opencode** под Python-монорепозиторий:
Python, Django, FastAPI, FastStream, aiogram и сквозные темы (безопасность, БД,
тестирование, документация, аудит).

Каждый навык — папка `skills/<name>/` с `SKILL.md` и `references/`. opencode
подгружает только `name` и `description`, а тело читает по требованию
(progressive disclosure).

## Навыки

| Навык | Назначение |
|---|---|
| `python` | язык: слои, типизация, async, ошибки, логи, тесты, доки, инструменты |
| `django` | Django 5.x: слои, ORM, selectors, forms, транзакции, миграции |
| `fastapi` | FastAPI: 4 слоя, read-only данные, DI, Pydantic v2, RFC 7807 |
| `pytest-bdd` | BDD/Gherkin, step definitions, Page Objects, отчёты, CI |
| `faststream` | FastStream 0.7.x + NATS: handlers, AckPolicy, идемпотентность |
| `aiogram` | aiogram 3.x: routers, FSM, flood-control, deploy |
| `security` | OWASP, auth, инъекции, секреты, security review |
| `postgres` | индексы, планы, JSONB, партиции, maintenance, репликация |
| `migration-safety` | безопасные Django-миграции (zero-downtime, backfill) |
| `api-design` | REST, пагинация, ошибки (RFC 7807), версионирование, OpenAPI |
| `python-testing` | pytest unit/integration, моки, async, антипаттерны |
| `documentation` | README, ADR, аудит доков |
| `git-commits` | атомарные коммиты, ветвление, Conventional Commits |
| `python-audit` | аудит готовности: статанализ, 8 испытаний, баллы |

Версии и история — в [CHANGELOG.md](CHANGELOG.md); план и решения — в
[ROADMAP.md](ROADMAP.md).

## Структура

```
skills/
├── python/
│   ├── SKILL.md
│   └── references/          # глубина по темам
├── django/
├── fastapi/
├── pytest-bdd/
├── faststream/
├── aiogram/
├── security/
├── postgres/
├── migration-safety/
├── api-design/
├── python-testing/
├── documentation/
├── git-commits/
└── python-audit/
```

## Подключение

Навыки — обычные каталоги `**/SKILL.md`. Варианты:

**1. Проектный конфиг** (`opencode.json`):

```json
{
  "$schema": "https://opencode.ai/config.json",
  "skills": { "paths": ["./skills"] }
}
```

**2. Глобально** — скопировать/симлинкнуть папки навыков в
`~/.config/opencode/skills/`.

После изменения конфига перезапусти opencode (конфиг читается один раз при старте).

## Как устроен навык

```
<name>/
├── SKILL.md              # имя файла ровно SKILL.md, совпадает с папкой
└── references/<topic>.md # по мере необходимости
```

- **Frontmatter:** только `name`, `description`, `license`, `compatibility`,
  `metadata` (прочие поля opencode игнорирует; `when_to_use` не существует).
- **`description`** (1–1024 символа): что делает + когда триггерить, с ключевыми
  словами; явно отсекать соседние темы.
- **`SKILL.md`** — сухие буллеты и таблицы, ≤ ~250 строк; глубина — в
  `references/`.
- **Правила** делятся на инварианты (всегда) и политики проекта (с обоснованием);
  конфликты источников разрешаются в пользу специфики проекта.

## Документы

- [ROADMAP.md](ROADMAP.md) — план, инвентаризация источников, фазы, DoD.
- [CHANGELOG.md](CHANGELOG.md) — версии навыков и история.
