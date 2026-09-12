---
name: postgres
description: >
  Use when diagnosing or optimizing PostgreSQL 15+ in a Python stack (Django,
  FastAPI, SQLAlchemy): reading EXPLAIN (ANALYZE, BUFFERS), indexing (B-tree,
  GIN, GiST, BRIN, partial, covering), query rewrites, pg_stat_statements,
  configuration tuning, connection pooling (pgbouncer), VACUUM/autovacuum/bloat,
  JSONB, replication and backups. Триггеры: PostgreSQL, Postgres, EXPLAIN,
  pg_stat_statements, slow query, индекс, JSONB, VACUUM, autovacuum, pgbouncer,
  replication, PITR. Безопасность миграций — migration-safety; N+1 — django/fastapi.
license: Proprietary
compatibility: opencode
metadata:
  version: "1.0.0"
  domain: database
  triggers: PostgreSQL, Postgres, EXPLAIN, pg_stat_statements, index, JSONB, VACUUM, autovacuum, pgbouncer, replication, slow query
  role: specialist
  scope: optimization
  output-format: analysis-and-code
  related-skills: django, fastapi, migration-safety, python
---

# PostgreSQL

Диагностика и оптимизация PostgreSQL 15+ в Python-стеке. Порядок всегда один:
**сначала найти виновника измерением, потом лечить** — никаких индексов и
настроек вслепую.

> N+1 и ORM-паттерны — в навыках `django`/`fastapi`. Создание индексов на живой
> таблице — через навык `migration-safety`.

## Когда применять

- Запрос/страница тормозит: разбор плана и индекса.
- «БД медленная»: поиск топ-запросов, тюнинг, пулинг.
- JSONB, расширения, репликация, бэкапы.

## Принципы

1. **Измерять, а не угадывать.** Сначала `pg_stat_statements`/лог, потом `EXPLAIN`.
2. **Одно изменение за раз**, эффект замерять тем же запросом.
3. **Read-only диагностика безопасна всегда**; изменения схемы/конфига — явный
   шаг с планом отката.
4. **Индекс — не бесплатный**: замедляет запись, ест место.
5. **Не выключать autovacuum.**

## Диагностика

```sql
-- включить один раз (требует рестарта)
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

SELECT query, calls, total_exec_time, mean_exec_time, rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;
```

Без расширения — `log_min_duration_statement = 500` (мс) и разбор лога.

```sql
EXPLAIN (ANALYZE, BUFFERS) <query>;
```

Красные флаги:

| В плане                                      | Причина               | Лечение                          |
| -------------------------------------------- | --------------------- | -------------------------------- |
| `Seq Scan` на большой таблице + узкий фильтр | нет индекса           | индекс по фильтру                |
| estimate `rows` ≫ actual                     | устаревшая статистика | `ANALYZE tbl`                    |
| `Sort Method: external merge Disk`           | сортировка не влезла  | `work_mem`/индекс под `ORDER BY` |
| `Nested Loop` с большим внешним набором      | плохой join-план      | индекс по join-колонке           |
| `Rows Removed by Filter` огромно             | не тот индекс         | составной/partial индекс         |
| высокие `Buffers: read` при повторе          | нет в кеше/bloat      | `shared_buffers`, vacuum         |

Подробно: [references/diagnostics.md](references/diagnostics.md).

## Индексы

- Тип: **B-tree** (равенство/диапазон/сортировка), **GIN** (JSONB, массивы,
  `tsvector`, `pg_trgm`), **GiST** (диапазоны, геометрия), **BRIN** (большие
  append-only).
- Порядок колонок: **равенство → диапазон → сортировка**; работает только левый
  префикс.
- **Partial** — для узкого горячего подмножества (`WHERE status='pending'`).
- **Expression** — точное совпадение выражения (`LOWER(email)`).
- **Covering** (`INCLUDE`) — для Index Only Scan.
- На живой таблице — только `CREATE INDEX CONCURRENTLY`.

```sql
CREATE INDEX CONCURRENTLY idx_orders_user_date
  ON orders(user_id, created_at DESC);
CREATE INDEX idx_orders_pending ON orders(user_id) WHERE status = 'pending';
```

Django: `AddIndexConcurrently` + `atomic = False`. SQLAlchemy: `Index(...)`,
`postgresql_where`.

Подробно: [references/indexes.md](references/indexes.md).

## Запросы

- Читать `EXPLAIN (ANALYZE, BUFFERS)` до и после.
- Плохие паттерны: `SELECT *`, функция на индексируемой колонке
  (`WHERE DATE(col)=...`), `OR` по колонкам, неявные касты, огромные `IN`,
  `LIKE '%term%'` без `pg_trgm`/FTS, большой `OFFSET`.
- Пагинация — **keyset**, не `OFFSET`.

```sql
-- keyset
SELECT * FROM products
WHERE (created_at, id) < ($1, $2)
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

Подробно: [references/queries.md](references/queries.md).

## JSONB

```sql
CREATE INDEX idx_docs_gin ON documents USING GIN (data);
SELECT * FROM documents WHERE data @> '{"status":"active"}';
```

- `JSONB`, не `json`; `@>` — индексируемый оператор.
- `jsonb_path_ops` — меньше/быстрее, только `@>`.
- Горячие скалярные поля — в generated-колонки с B-tree.
- Большие массивы/часто обновляемые поля — не в JSONB.

Подробно: [references/jsonb.md](references/jsonb.md).

## Конфигурация и пулинг

Ориентиры (VPS):

- `shared_buffers` ≈ 25% RAM; `effective_cache_size` ≈ 50–75% RAM.
- `work_mem` — осторожно (× сортировок × соединений); тяжёлый запрос —
  `SET LOCAL work_mem='64MB'`.
- `maintenance_work_mem` выше на время `VACUUM`/`CREATE INDEX`.
- `random_page_cost = 1.1` на SSD.
- `max_connections` умеренно + **pgbouncer** (transaction mode) спереди.
- `statement_timeout`, `idle_in_transaction_session_timeout`.

Django: `CONN_MAX_AGE`; SQLAlchemy: `pool_size`/`max_overflow` ≤ `max_connections`.

Подробно: [references/tuning-maintenance.md](references/tuning-maintenance.md).

## Maintenance

- Autovacuum — включён; для горячих таблиц понизить `scale_factor` per-table.
- `VACUUM (ANALYZE)` — не блокирует; `VACUUM FULL` — эксклюзивная блокировка
  (на проде — окно или `pg_repack`).
- `REINDEX INDEX CONCURRENTLY` (PG12+) — без блокировки.
- Мониторинг: `pg_stat_user_tables` (dead tuples), `pg_stat_activity`
  (долгие/idle-in-transaction), `pg_locks`, возраст XID (wraparound).

Подробно: [references/tuning-maintenance.md](references/tuning-maintenance.md).

## Партиционирование

Для больших/растущих таблиц (time-series, архивация). Range/List/Hash; pruning
работает по ключу партиционирования; уникальные ключи должны включать partition
key; старые партиции — `DETACH`, не массовый `DELETE`.

Подробно: [references/partitioning.md](references/partitioning.md).

## Репликация и бэкапы

- **Physical/streaming** — HA/read-реплики (байт-в-байт).
- **Logical** — выборочная репликация, кросс-версии.
- **Replication slots** — предотвращают удаление WAL; следить, чтобы не забили диск.
- Failover: `pg_promote()`; автоматизация — Patroni.
- Бэкапы: `pg_dump`/`pg_basebackup` + WAL-архив + **PITR**; рестор тестировать.

Подробно: [references/replication-backups.md](references/replication-backups.md).

## Расширения

`pg_stat_statements` (обязательно), `pg_trgm` (`ILIKE '%...%'`, fuzzy),
`gen_random_uuid()` (PG13+, без расширения), `citext`, `btree_gin`, PostGIS.

Подробно: [references/extensions.md](references/extensions.md).

## Уровни риска изменений

| Действие                               | Риск                         |
| -------------------------------------- | ---------------------------- |
| `EXPLAIN`, `pg_stat_*`                 | безопасно всегда             |
| `CREATE INDEX CONCURRENTLY`, `ANALYZE` | безопасно, но I/O — не в пик |
| `postgresql.conf`, pgbouncer           | рестарт/перезагрузка — окно  |
| `VACUUM FULL`, failover, schema change | окно + план отката           |

## Запрещённые паттерны

| ❌ Запрещено                     | ✅ Правильно                  |
| -------------------------------- | ----------------------------- |
| Индексы/настройки без замера     | измерение → изменение → замер |
| `SELECT *` в проде               | нужные колонки                |
| `CREATE INDEX` на живой таблице  | `CONCURRENTLY`                |
| Функция на индексируемой колонке | диапазонный предикат          |
| `LIKE '%term%'` без индекса      | `pg_trgm`/FTS                 |
| Большой `OFFSET`                 | keyset-пагинация              |
| `json` вместо `jsonb`            | `jsonb` + GIN                 |
| `VACUUM FULL` в рабочие часы     | окно/`pg_repack`              |
| Отключение autovacuum            | оставить включённым           |
| Много `max_connections`          | пулинг (pgbouncer)            |
| Несколько изменений сразу        | по одному, с замером          |
| Хранить большие BLOB в БД        | объектное хранилище           |

## Чек-лист

- [ ] Виновник найден измерением, а не догадкой.
- [ ] План прочитан: estimate ≈ actual, нет лишних `Seq Scan`.
- [ ] Индекс селективен, порядок колонок обоснован, `CONCURRENTLY`.
- [ ] Проверены неиспользуемые/дублирующие индексы.
- [ ] Пагинация keyset, нет больших `OFFSET`.
- [ ] JSONB индексирован; горячие поля вынесены.
- [ ] `shared_buffers`/`effective_cache_size` под RAM; `work_mem` осторожно.
- [ ] Пул соединений настроен.
- [ ] Autovacuum включён; bloat и долгие транзакции под контролем.
- [ ] Бэкапы и рестор протестированы; lag репликации мониторится.
- [ ] Изменения на проде — явным шагом с планом отката.

## Справочники

| Тема                 | Reference                                                              | Загружать когда                         |
| -------------------- | ---------------------------------------------------------------------- | --------------------------------------- |
| Диагностика, EXPLAIN | [references/diagnostics.md](references/diagnostics.md)                 | Поиск медленных запросов, чтение плана  |
| Индексы              | [references/indexes.md](references/indexes.md)                         | Проектирование индексов                 |
| Запросы              | [references/queries.md](references/queries.md)                         | Переписывание запросов, пагинация       |
| Конфиг и maintenance | [references/tuning-maintenance.md](references/tuning-maintenance.md)   | Память, пулинг, VACUUM, bloat           |
| JSONB                | [references/jsonb.md](references/jsonb.md)                             | JSONB-операторы, GIN, generated-колонки |
| Партиционирование    | [references/partitioning.md](references/partitioning.md)               | Большие/растущие таблицы, архивация     |
| Репликация и бэкапы  | [references/replication-backups.md](references/replication-backups.md) | HA, логическая репликация, PITR         |
| Расширения           | [references/extensions.md](references/extensions.md)                   | pg_trgm, pgcrypto, PostGIS              |

## Связанные навыки

- `django` / `fastapi` — ORM, N+1, `select_related`/`prefetch_related`,
  `joinedload`/`selectinload`.
- `migration-safety` — безопасное создание индексов/constraint.
- `python` — общие практики, инструменты.
