# docker-compose

Все сервисы докеризуются — и для разработки, и для продакшена. Общий каркас
сервиса выносится в YAML-anchor (`x-` extension).

## Anchor общего сервиса

```yaml
x-app:
  &app
  build:
    context: .
    dockerfile: ./Dockerfile
    target: develop            # dev; для prod — production
  tty: true
  restart: unless-stopped
  volumes:
    - .:/opt/app:cached

services:
  postgresql:
    image: postgres:18-alpine
    container_name: app-postgresql
    restart: unless-stopped
    volumes: [app-postgres-data:/var/lib/postgresql]
    environment:
      - POSTGRES_DB=app_data
      - POSTGRES_USER=app_user
      - POSTGRES_PASSWORD=app_pass
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -d app_data -U app_user"]
      interval: 15s
      timeout: 10s
      retries: 10
    ports: ["5432:5432"]

  django:
    <<: *app
    container_name: app-main
    ports: ["8000:8000"]
    volumes:
      - .:/opt/app:cached
      - ../bdd/data:/opt/app/bdd-data:cached
    environment:
      - BDD_DATA_DIR=/opt/app/bdd-data
    command: uv run python manage.py runserver 0.0.0.0:8000
    depends_on:
      postgresql: { condition: service_healthy }
    develop:
      watch:
        - action: sync
          path: ./

volumes:
  app-postgres-data:
```

## Паттерны

- **Anchor `x-app` + `<<: *app`** — общий build/restart/tty/volumes; сервис
  добавляет свои `ports`/`command`/`environment`.
- **`target: develop`** для локальной разработки; в проде — `production`.
- **Bind-mount `.:/opt/app:cached`** — код с хоста (dev); в prod код в образе.
- **`develop.watch`** — `docker compose up --watch` синхронизирует изменения.
- **Healthcheck + `depends_on: condition: service_healthy`** — БД готова до старта.
- **Именованные volumes** для данных (`app-postgres-data`).
- **Monorepo**: монтирование корня и соседних каталогов (`../bdd/data`).

## Несколько сервисов (монорепозиторий)

```yaml
  fastapi:
    <<: *app
    command: uv run uvicorn app.main:app --host 0.0.0.0 --port 8080
    ports: ["8080:8080"]

  payment_listener:
    <<: *app
    command: uv run python manage.py run_payment_listener
    depends_on:
      postgresql: { condition: service_healthy }

  metrics_exporter:
    <<: *app
    command: uv run python manage.py export_metrics
```

- Web, воркеры, listener, exporter — отдельные сервисы с общим anchor/образом.
- Один `postgresql`/`redis` на все сервисы (общая сеть compose).

## Env и секреты

- `.env` — в `.gitignore`; `.env.example` — без значений.
- В compose — `environment:`/`env_file:`; в проде — Docker secrets.
- Пароли БД в примере — только для локальной разработки.

## Прод-вариант

```bash
docker compose -f docker-compose.yaml -f docker-compose.prod.yaml up -d
```

- `target: production`; без bind-mount и `--reload`.
- Секреты — secrets/`EnvironmentFile`; ресурсы/реплики — в override-файле.

## Антипаттерны

| ❌ | ✅ |
|---|---|
| дублирование конфига в каждом сервисе | anchor `x-app` |
| `image: postgres` без версии | `postgres:18-alpine` |
| секреты в `environment:` | `env_file`/secrets |
| `depends_on` без condition | `condition: service_healthy` |
| bind-mount `.` в прод | код в образе |
| один сервис на всё | web/worker/listener раздельно |

## Чек-лист

- [ ] Anchor для общего сервиса; сервисы через `<<: *app`.
- [ ] `target: develop` в dev, `production` в prod.
- [ ] Версии образов пинованы; healthcheck у БД.
- [ ] `depends_on` с `service_healthy`.
- [ ] Данные — в именованных volumes.
- [ ] Секреты — `env_file`/secrets; `.env` в `.gitignore`.
- [ ] `develop.watch` для dev; в prod — без bind-mount.
