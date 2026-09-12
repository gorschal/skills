---
name: docker
description: >
  Use when writing or reviewing Docker and docker-compose for services:
  multi-stage Dockerfile, non-root user, image size and caching, .dockerignore,
  healthchecks, compose services/volumes/env, dev watch. Триггеры: Docker,
  Dockerfile, docker-compose, compose, image, multi-stage, .dockerignore,
  healthcheck, volume, container. Деплой под systemd/nginx — вне scope.
license: Proprietary
compatibility: opencode
metadata:
  version: "1.1.0"
  domain: devops
  triggers: Docker, Dockerfile, docker-compose, compose, image, multi-stage, .dockerignore, healthcheck, container
  role: specialist
  scope: implementation
  output-format: code
  related-skills: python, django, fastapi, postgres
---

# Docker

Dockerfile и docker-compose для сервисов монорепозитория (Django, FastAPI,
воркеры, Postgres, Redis). Принцип: маленький безопасный образ + предсказуемый
локальный запуск.

## Когда применять

- Dockerfile, docker-compose, `.dockerignore`.
- Уменьшение образа, кэш слоёв, non-root, healthcheck.
- Локальный dev-запуск сервисов и зависимостей.

## Ключевые принципы

1. **3 стадии** `develop`/`prerelease`/`production`: рантайм — минимальный.
2. **Non-root** пользователь в рантайме.
3. **Пин версий** базовых образов (не `latest`).
4. **`.dockerignore`** — исключать `.git`, `.env`, кэши, тесты, доки.
5. **Секреты — в рантайме** (env/secret), не в образе.
6. **Healthcheck** для каждого долгоживущего сервиса.
7. **Кэш слоёв**: сначала манифесты зависимостей, потом код.

## Dockerfile (3 стадии)

`develop` → `prerelease` → `production`, база `python:3.12-slim-bookworm`.

```dockerfile
# syntax=docker/dockerfile:1
FROM python:3.12-slim-bookworm AS develop
ARG UID=1000
ARG GID=1000
ENV PROJECTPATH=/opt/app UV_LINK_MODE=copy UV_COMPILE_BYTECODE=1 PYTHONUNBUFFERED=1
RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt/lists,sharing=locked \
    apt-get update && apt-get install -y --no-install-recommends libpq-dev gcc git
RUN groupadd -g "$GID" appuser && useradd -u "$UID" -g "$GID" -m appuser
COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /usr/local/bin/
USER appuser
WORKDIR "$PROJECTPATH"
COPY --chown=appuser:appuser pyproject.toml uv.lock ./
RUN --mount=type=cache,target=/home/appuser/.cache/uv,uid=1000,gid=1000 \
    uv sync --no-install-project
# prerelease: uv sync --frozen --no-dev + collectstatic (dummy secret)
# production: только libpq5; копия .venv+кода из prerelease; EXPOSE 8000
```

- `develop` (dev-зависимости) → `prerelease` (`--no-dev` + `collectstatic`) → `production` (runtime-only).
- Non-root `appuser` с UID/GID хоста; `--chown`.
- BuildKit-кэши apt/uv; `uv` из официального образа.

Подробно: [references/dockerfile.md](references/dockerfile.md).

## docker-compose

Общий каркас сервиса — YAML-anchor `x-app` + `<<: *app`; всё докеризуется (dev и prod).

```yaml
x-app: &app
  build: { context: ., dockerfile: ./Dockerfile, target: develop }
  tty: true
  restart: unless-stopped
  volumes: [".:/opt/app:cached"]

services:
  postgresql:
    image: postgres:18-alpine
    volumes: [pgdata:/var/lib/postgresql]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app_user"]
      interval: 15s
      timeout: 10s
      retries: 10

  django:
    <<: *app
    ports: ["8000:8000"]
    command: uv run python manage.py runserver 0.0.0.0:8000
    depends_on:
      postgresql: { condition: service_healthy }
    develop:
      watch:
        - action: sync
          path: ./

volumes:
  pgdata:
```

- Anchor для общего build/restart/volumes; сервис добавляет свои поля.
- `target: develop` в dev, `production` в prod.
- `develop.watch` + bind-mount — локальная разработка; в prod код в образе.
- `depends_on: condition: service_healthy`; именованные volumes; monorepo-монтирование.

Подробно: [references/compose.md](references/compose.md).

## Безопасность

- Non-root пользователь; минимальный базовый образ (`-slim`/`-alpine`).
- Не класть секреты в образ/слои; `.env` — в `.dockerignore`.
- Пиновать версии; сканировать образы (`docker scout`, `trivy`).
- Не монтировать docker socket в контейнер без необходимости.
- Read-only rootfs и `cap_drop: [ALL]` — где возможно.

## Запрещённые паттерны

| ❌                                               | ✅                             |
| ------------------------------------------------ | ------------------------------ |
| `FROM image:latest`                              | пин версии                     |
| `USER root` в рантайме                           | non-root                       |
| Секреты в `ENV`/`COPY .env`                      | runtime env/secret             |
| Один слой с `COPY . .` до установки зависимостей | сначала манифесты              |
| Нет `.dockerignore`                              | исключать `.git`, `.env`, кэши |
| `apt-get upgrade`/лишние пакеты                  | только необходимое             |
| Данные в слое контейнера                         | именованный volume             |
| Нет healthcheck                                  | `HEALTHCHECK`                  |

## Чек-лист

- [ ] Multi-stage; рантайм-образ минимальный.
- [ ] Non-root `USER`; `--chown` для файлов.
- [ ] Базовые образы и зависимости пинованы.
- [ ] `.dockerignore` исключает `.git`, `.env`, кэши, тесты.
- [ ] Секреты — рантайм, не в образе.
- [ ] `HEALTHCHECK` есть; `depends_on` учитывает health.
- [ ] Данные — в именованных volumes.
- [ ] Образ сканируется (`trivy`/`docker scout`).

## Справочники

| Тема           | Reference                                            | Когда                                |
| -------------- | ---------------------------------------------------- | ------------------------------------ |
| Dockerfile     | [references/dockerfile.md](references/dockerfile.md) | Multi-stage, кэш, non-root, размер   |
| docker-compose | [references/compose.md](references/compose.md)       | Сервисы, volumes, env, health, watch |

## Связанные навыки

- `python` / `django` / `fastapi` — сервисы в контейнерах.
- `postgres` — БД как сервис compose.
- `security` — секреты, сканирование образов.
