# Dockerfile

BuildKit: первая строка — `# syntax=docker/dockerfile:1`. База —
`python:3.12-slim-bookworm`. Три стадии: **develop → prerelease → production**.

## Стадии

| Стадия       | Назначение                            | Зависимости                           |
| ------------ | ------------------------------------- | ------------------------------------- |
| `develop`    | локальная разработка, полная сборка   | `libpq-dev gcc git` + dev-зависимости |
| `prerelease` | без dev-зависимостей, `collectstatic` | `libpq-dev gcc git`                   |
| `production` | минимальный рантайм                   | только `libpq5`                       |

```dockerfile
# syntax=docker/dockerfile:1

FROM python:3.12-slim-bookworm AS develop
ARG UID=1000
ARG GID=1000
ENV PROJECTPATH=/opt/app \
    UV_COMPILE_BYTECODE=1 UV_LINK_MODE=copy UV_PYTHON_DOWNLOADS=0 \
    PYTHONFAULTHANDLER=1 PYTHONUNBUFFERED=1
RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt/lists,sharing=locked \
    apt-get update && apt-get install -y --no-install-recommends libpq-dev gcc git \
    && rm -rf /var/lib/apt/lists/*
RUN groupadd -g "${GID}" appuser \
    && useradd -u "${UID}" -g "${GID}" -m appuser \
    && mkdir -p "${PROJECTPATH}" && chown -R appuser:appuser "${PROJECTPATH}"
COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /usr/local/bin/
USER appuser
WORKDIR "${PROJECTPATH}"
COPY --chown=appuser:appuser pyproject.toml uv.lock ./
RUN --mount=type=cache,target=/home/appuser/.cache/uv,sharing=locked,uid=1000,gid=1000 \
    uv sync --no-install-project

FROM python:3.12-slim-bookworm AS prerelease
ARG UID=1000
ARG GID=1000
ENV PROJECTPATH=/opt/app \
    UV_COMPILE_BYTECODE=1 UV_LINK_MODE=copy UV_PYTHON_DOWNLOADS=0 \
    PYTHONFAULTHANDLER=1 PYTHONUNBUFFERED=1
RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt/lists,sharing=locked \
    apt-get update && apt-get install -y --no-install-recommends libpq-dev gcc git \
    && rm -rf /var/lib/apt/lists/*
RUN groupadd -g "${GID}" appuser \
    && useradd -u "${UID}" -g "${GID}" -m appuser \
    && mkdir -p /opt/static && chown -R appuser:appuser /opt/static
COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /usr/local/bin/
USER appuser
WORKDIR "${PROJECTPATH}"
COPY --from=develop --chown=appuser:appuser "${PROJECTPATH}/pyproject.toml" ./
COPY --from=develop --chown=appuser:appuser "${PROJECTPATH}/uv.lock" ./
RUN --mount=type=cache,target=/home/appuser/.cache/uv,sharing=locked,uid=1000,gid=1000 \
    uv sync --frozen --no-dev
COPY --chown=appuser:appuser . ./
RUN DJANGO_SECRET_KEY=dummy uv run python manage.py collectstatic --no-input

FROM python:3.12-slim-bookworm AS production
ARG UID=1000
ARG GID=1000
LABEL org.opencontainers.image.title="app"
ENV PROJECTPATH=/opt/app \
    PATH="/opt/app/.venv/bin:${PATH}" \
    PYTHONFAULTHANDLER=1 PYTHONUNBUFFERED=1
RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt/lists,sharing=locked \
    apt-get update && apt-get install -y --no-install-recommends libpq5 \
    && rm -rf /var/lib/apt/lists/* \
    && groupadd -g "${GID}" appuser && useradd -u "${UID}" -g "${GID}" -m appuser
USER appuser
WORKDIR "${PROJECTPATH}"
COPY --from=prerelease --chown=appuser:appuser "${PROJECTPATH}" "${PROJECTPATH}"
COPY --from=prerelease --chown=appuser:appuser /opt/static /opt/static
EXPOSE 8000
```

## Non-root с UID/GID хоста

`ARG UID/GID` (по умолчанию 1000) + `groupadd`/`useradd` → `appuser`. Согласование
с хостом избегает root-owned файлов при bind-mount в dev. **ARG/ENV скоупятся по
стадии** — объявлять их в каждой стадии, где используются.

## BuildKit-кэши

- apt: `--mount=type=cache,target=/var/cache/apt` + `/var/lib/apt/lists`.
- uv: `--mount=type=cache,target=/home/appuser/.cache/uv,uid=1000,gid=1000`.

## uv

Ставится копированием из `ghcr.io/astral-sh/uv:latest` (без pip). Синхронизация:

- develop: `uv sync --no-install-project` (код копируется позже).
- prerelease: `uv sync --frozen --no-dev`.
- `UV_LINK_MODE=copy`, `UV_COMPILE_BYTECODE=1`, `UV_PYTHON_DOWNLOADS=0`.

## Runtime vs build

- Build-стадии: `libpq-dev gcc git`.
- Production: только `libpq5` (runtime), без компиляторов.
- `collectstatic` — в prerelease с dummy-секретом (`DJANGO_SECRET_KEY=dummy`).

## .dockerignore

```
.git
.env*
.venv
__pycache__
*.pyc
.pytest_cache
.ruff_cache
node_modules
coverage
docs
*.md
Dockerfile*
docker-compose*
.dockerignore
```

## Антипаттерны

| ❌                                           | ✅                                         |
| -------------------------------------------- | ------------------------------------------ |
| `FROM python:latest`                         | `python:3.12-slim-bookworm`                |
| root в рантайме                              | `appuser` (UID/GID хоста)                  |
| build-tools в production                     | только runtime (`libpq5`)                  |
| секреты в `ENV`                              | runtime env/`.env`                         |
| `COPY . .` до `uv sync`                      | сначала `pyproject.toml`/`uv.lock`         |
| нет кэш-маунтов                              | BuildKit cache для apt/uv                  |
| `ARG`/`ENV` объявлены только в первой стадии | в каждой стадии, где нужны                 |
| один образ на dev/prod                       | стадии `develop`/`prerelease`/`production` |

## Чек-лист

- [ ] Три стадии: `develop`/`prerelease`/`production`.
- [ ] `ARG UID/GID` и `ENV PROJECTPATH` объявлены в каждой используемой стадии.
- [ ] Non-root `appuser` с UID/GID; `--chown`.
- [ ] BuildKit-кэши для apt и uv.
- [ ] `uv sync --frozen --no-dev` в prerelease; runtime-only в production.
- [ ] `collectstatic` с dummy-секретом.
- [ ] Секретов в образе нет; `.dockerignore` есть.
