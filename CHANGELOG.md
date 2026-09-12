# CHANGELOG

Версии навыков — в `metadata.version` каждого `SKILL.md`.

Схема версионирования (semver):

- **major** — ломающее изменение правил/структуры навыка;
- **minor** — новые разделы, references, правила;
- **patch** — правки формулировок, ссылок, опечаток.

## Навыки

| Навык              | Версия | Назначение                                                           |
| ------------------ | ------ | -------------------------------------------------------------------- |
| `python`           | 1.2.0  | язык: слои, типизация, async, ошибки, логи, тесты, доки, инструменты |
| `django`           | 1.1.0  | Django 5.x: слои, ORM, selectors, forms, транзакции, миграции        |
| `fastapi`          | 1.1.0  | FastAPI: 4 слоя, read-only данные, DI, Pydantic v2, RFC 7807         |
| `pytest-bdd`       | 1.0.0  | BDD/Gherkin, step definitions, Page Objects, отчёты, CI              |
| `faststream`       | 1.0.0  | FastStream 0.7.x + NATS: handlers, AckPolicy, идемпотентность        |
| `aiogram`          | 1.0.0  | aiogram 3.x: routers, FSM, flood-control, deploy                     |
| `security`         | 1.0.0  | OWASP, auth, инъекции, секреты, security review                      |
| `postgres`         | 1.0.0  | индексы, планы, JSONB, партиции, maintenance, репликация             |
| `migration-safety` | 1.0.0  | безопасные Django-миграции (zero-downtime, backfill)                 |
| `api-design`       | 1.0.0  | REST, пагинация, ошибки (RFC 7807), версионирование, OpenAPI         |
| `python-testing`   | 1.0.0  | pytest unit/integration, моки, async, антипаттерны                   |
| `documentation`    | 1.2.0  | README, ARCHITECTURE.md, ADR, AGENTS.md/CLAUDE.md, аудит доков      |
| `git-commits`      | 1.0.0  | атомарные коммиты, ветвление, Conventional Commits                   |
| `python-audit`     | 1.0.0  | аудит готовности: статанализ, 8 испытаний, баллы                     |
| `solidity`         | 1.2.0  | Solidity + Foundry: безопасность, газ, UUPS, тесты/аудит, NatSpec-стиль |
| `docker`           | 1.1.0  | Dockerfile (3 стадии), docker-compose (anchors, watch)               |
| `javascript`       | 1.0.0  | базовые принципы JS/Node (ES2023+, async, ESM)                       |
| `php`              | 1.0.0  | базовые принципы PHP (PSR-12, слои, безопасность, платформы)         |

## История

- **2026-09-12** — `solidity` 1.2.0: добавлен стиль NatSpec
  (`references/natspec.md`) — форматирование `///` и `/** */`, границы
  применения; новые триггеры в description.
- **2026-09-11** — `documentation` 1.2.0: добавлен ARCHITECTURE.md
  (`references/architecture.md`) — устройство системы, границы, потоки.
- **2026-09-11** — `documentation` 1.1.0: добавлен AGENTS.md/CLAUDE.md
  (`references/agents.md`).
- **2026-09-11** — `solidity` 1.1.0, `docker` 1.1.0: приведены к реальным
  конвенциям (Foundry: `tests/`/`scripts/`/`deployments/`, anvil-форк; Docker:
  стадии `develop`/`prerelease`/`production`, anchors + `develop.watch`).
