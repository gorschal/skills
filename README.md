# skills

[![opencode](https://img.shields.io/badge/-opencode-000000?style=flat)](https://opencode.ai)
[![Python](https://img.shields.io/badge/-Python_3.12-3776AB?style=flat&logo=python&logoColor=white)](https://www.python.org/)
[![License](https://img.shields.io/badge/-Proprietary-blue?style=flat)](LICENSE.txt)

Набор навыков (agent skills) для **opencode** под Python-монорепозиторий:
Python, Django, FastAPI, FastStream, aiogram и сквозные темы (безопасность, БД,
тестирование, документация, аудит).

Каждый навык — папка `skills/<name>/` с `SKILL.md` и `references/`. opencode
подгружает только `name` и `description`, а тело читает по требованию
(progressive disclosure).

## Навыки

| Навык              | Назначение                                                           |
| ------------------ | -------------------------------------------------------------------- |
| `python`           | язык: слои, типизация, async, ошибки, логи, тесты, доки, инструменты |
| `django`           | Django 5.x: слои, ORM, selectors, forms, транзакции, миграции        |
| `fastapi`          | FastAPI: 4 слоя, read-only данные, DI, Pydantic v2, RFC 7807         |
| `pytest-bdd`       | BDD/Gherkin, step definitions, Page Objects, отчёты, CI              |
| `faststream`       | FastStream 0.7.x + NATS: handlers, AckPolicy, идемпотентность        |
| `aiogram`          | aiogram 3.x: routers, FSM, flood-control, deploy                     |
| `security`         | OWASP, auth, инъекции, секреты, security review                      |
| `postgres`         | индексы, планы, JSONB, партиции, maintenance, репликация             |
| `migration-safety` | безопасные Django-миграции (zero-downtime, backfill)                 |
| `api-design`       | REST, пагинация, ошибки (RFC 7807), версионирование, OpenAPI         |
| `python-testing`   | pytest unit/integration, моки, async, антипаттерны                   |
| `documentation`    | README, ARCHITECTURE.md, ADR, AGENTS.md/CLAUDE.md, аудит доков       |
| `git-commits`      | атомарные коммиты, ветвление, Conventional Commits                   |
| `python-audit`     | аудит готовности: статанализ, 8 испытаний, баллы                     |
| `solidity`         | Solidity + Foundry: безопасность, газ, UUPS, тесты/аудит             |
| `docker`           | Dockerfile (multi-stage, non-root), docker-compose                   |
| `javascript`       | базовые принципы JS/Node (ES2023+, async, ESM)                       |
| `php`              | базовые принципы PHP (PSR-12, слои, безопасность, платформы)         |

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
├── python-audit/
├── solidity/
├── docker/
├── javascript/
└── php/
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

## Плагины (опционально)

Плагины — это JS/TS, который выполняется при старте opencode и может
трансформировать промпт/инструменты. Ставить только доверенные, фиксировать
версии, начинать с минимума. Каталог — [opencode.ai/docs/ecosystem](https://opencode.ai/docs/ecosystem).

**Рекомендуемые:**

| Плагин                             | Зачем                                                                              |
| ---------------------------------- | ---------------------------------------------------------------------------------- |
| `opencode-vibeguard`               | Редактирует секреты/PII в плейсхолдеры до отправки в LLM, восстанавливает локально |
| `opencode-dynamic-context-pruning` | Чистит устаревшие tool-output'ы → экономия токенов                                 |
| `opencode-websearch-cited`         | Нативный веб-поиск со ссылками                                                     |
| `opencode-notify`                  | Уведомления о завершении/ошибках (в desktop-приложении уже есть)                   |

**Ситуативно:** `opencode-shell-strategy` (защита от TTY-зависаний),
`opencode-pty` (долгоживущие процессы), `opencode-worktree` (git worktree),
`opencode-scheduler` (регулярные задачи), `opencode-supermemory` (память между
сессиями), `@plannotator/opencode` (ревью планов), `opencode-firecrawl`/
`opencode-tavily` (веб-скрейпинг), `opencode-sentry-monitor` (Sentry).

**Не нужно:** `opencode-triage`/`opencode-skillful` — opencode уже лениво грузит
тела навыков, экономится лишь список описаний (при 14 навыках выгода скромная);
`oh-my-opencode` — большой бандл, брать только целиком осознанно.

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "lsp": true,
  "plugin": ["opencode-vibeguard", "opencode-dynamic-context-pruning", "opencode-websearch-cited"],
}
```

Имена npm-пакетов могут отличаться от имён репозиториев — сверять на npm и
просматривать исходники перед установкой.

## Документы

- [CHANGELOG.md](CHANGELOG.md) — версии навыков и история.
