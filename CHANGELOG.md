# CHANGELOG

Версии навыков — в `metadata.version` каждого `SKILL.md`.

Схема версионирования (semver):
- **major** — ломающее изменение правил/структуры навыка;
- **minor** — новые разделы, references, правила;
- **patch** — правки формулировок, ссылок, опечаток.

## Навыки

| Навык | Версия | Назначение |
|---|---|---|
| `python` | 1.2.0 | язык: слои, типизация, async, ошибки, логи, тесты, доки, инструменты |
| `django` | 1.1.0 | Django 5.x: слои, ORM, selectors, forms, транзакции, миграции |
| `fastapi` | 1.1.0 | FastAPI: 4 слоя, read-only данные, DI, Pydantic v2, RFC 7807 |
| `pytest-bdd` | 1.0.0 | BDD/Gherkin, step definitions, Page Objects, отчёты, CI |
| `faststream` | 1.0.0 | FastStream 0.7.x + NATS: handlers, AckPolicy, идемпотентность |
| `aiogram` | 1.0.0 | aiogram 3.x: routers, FSM, flood-control, deploy |
| `security` | 1.0.0 | OWASP, auth, инъекции, секреты, security review |
| `postgres` | 1.0.0 | индексы, планы, JSONB, партиции, maintenance, репликация |
| `migration-safety` | 1.0.0 | безопасные Django-миграции (zero-downtime, backfill) |
| `api-design` | 1.0.0 | REST, пагинация, ошибки (RFC 7807), версионирование, OpenAPI |
| `python-testing` | 1.0.0 | pytest unit/integration, моки, async, антипаттерны |
| `documentation` | 1.0.0 | README, ADR, аудит доков |
| `git-commits` | 1.0.0 | атомарные коммиты, ветвление, Conventional Commits |
| `python-audit` | 1.0.0 | аудит готовности: статанализ, 8 испытаний, баллы |

## История

- **2026-09-11** — Фаза 6: «caveman»-проход по всем SKILL.md (≤ ~250 строк),
  сквозная проверка на противоречия, RFC 7807 как единый формат ошибок,
  владение схемой — Django, BDD-first тест-политика. Навыки `python` (1.2.0),
  `django` (1.1.0), `fastapi` (1.1.0).
