# Конфигурация, пулинг и maintenance

## Память и планировщик (ориентиры, PG15+)

- `shared_buffers` ≈ 25% RAM (до 40% на выделенном хосте).
- `effective_cache_size` ≈ 50–75% RAM (подсказка планировщику, не аллокация).
- `work_mem` — на сортировку/хеш **на операцию и соединение**; умножается на
  число сортировок × соединений. Глобально скромно; тяжёлый запрос —
  `SET LOCAL work_mem='64MB'`.
- `maintenance_work_mem` 1–2 GB для `VACUUM`/`CREATE INDEX`; `autovacuum_work_mem`
  отдельно.
- `random_page_cost = 1.1` на SSD; `effective_io_concurrency` выше (~200).

```sql
ALTER SYSTEM SET shared_buffers = '4GB';            -- ~25% от 16GB
ALTER SYSTEM SET effective_cache_size = '12GB';     -- ~75%
ALTER SYSTEM SET work_mem = '40MB';
ALTER SYSTEM SET maintenance_work_mem = '2GB';
ALTER SYSTEM SET random_page_cost = 1.1;
ALTER SYSTEM SET max_parallel_workers_per_gather = 4;
ALTER SYSTEM SET statement_timeout = '30s';
ALTER SYSTEM SET idle_in_transaction_session_timeout = '5min';
```

Менять по одному параметру и замерять.

## Соединения и пулинг

- `max_connections` умеренно; масштабировать **пулером**, а не числом соединений.
- Симптом `too many connections` — сначала найти, кто держит
  (`pg_stat_activity`, `state='idle'`).
- Django: `CONN_MAX_AGE` достаточно для одного сервиса; pgbouncer (transaction
  mode) — когда воркеров/сервисов много.
- SQLAlchemy: `pool_size` + `max_overflow` ≤ `max_connections`.
- В transaction mode недоступны session-фичи: `SET`, advisory locks, prepared
  statements (Django — `server_side_binding` осторожно).

```ini
# pgbouncer
[pgbouncer]
pool_mode = transaction
max_client_conn = 1000
default_pool_size = 25
reserve_pool_size = 5
```

## Autovacuum и bloat

- MVCC оставляет мёртвые кортежи; VACUUM их освобождает и предотвращает
  XID-wraparound.
- Autovacuum держать **включённым**; для горячих таблиц понизить scale factor.
- `VACUUM` (обычный) не блокирует; `VACUUM FULL` — эксклюзивная блокировка и
  перезапись (на проде — окно или `pg_repack`).
- `REINDEX INDEX CONCURRENTLY` (PG12+) — без блокировки.
- `ANALYZE` после массовых изменений; поднять statistics target для искажённых
  колонок.

```sql
VACUUM (ANALYZE, VERBOSE) users;
REINDEX INDEX CONCURRENTLY idx_users_email;

ALTER TABLE orders SET (
  autovacuum_vacuum_scale_factor = 0.05,
  autovacuum_vacuum_threshold = 1000,
  autovacuum_analyze_scale_factor = 0.02
);

ALTER TABLE users ALTER COLUMN email SET STATISTICS 1000;
```

## Мониторинг

```sql
-- bloat / мёртвые кортежи
SELECT relname, n_dead_tup, n_live_tup, last_autovacuum,
       round(100.0*n_dead_tup/NULLIF(n_live_tup+n_dead_tup,0),2) AS dead_pct
FROM pg_stat_user_tables ORDER BY n_dead_tup DESC;

-- долгие запросы
SELECT pid, now()-query_start AS duration, state, query
FROM pg_stat_activity
WHERE state='active' AND now()-query_start > interval '5 minutes'
ORDER BY duration DESC;

-- idle in transaction (держат snapshot, мешают vacuum)
SELECT pid, now()-state_change AS idle_duration, query
FROM pg_stat_activity
WHERE state='idle in transaction' AND now()-state_change > interval '1 minute';

-- блокировки
SELECT l.pid, a.query, l.mode, l.granted, pg_blocking_pids(l.pid) AS blocked_by
FROM pg_locks l JOIN pg_stat_activity a ON a.pid = l.pid
WHERE NOT l.granted;

-- cache hit ratio
SELECT round(100.0*blks_hit/NULLIF(blks_hit+blks_read,0),2) AS hit_ratio
FROM pg_stat_database WHERE datname = current_database();

-- wraparound (age должен быть далеко от 1B)
SELECT datname, age(datfrozenxid) AS xid_age FROM pg_database ORDER BY 2 DESC;
```

Периодичность: ежедневно — autovacuum, долгие запросы, cache hit, lag;
еженедельно — медленные запросы, bloat, неиспользуемые индексы; ежемесячно —
autovacuum review, reindex; ежеквартально — тест рестора и обновления.

## Антипаттерны

| ❌ | ✅ |
|---|---|
| Отключение autovacuum | включён + per-table тюнинг |
| `VACUUM FULL` в рабочие часы | окно/`pg_repack` |
| Много `max_connections` | пулер |
| `work_mem` огромный глобально | `SET LOCAL` для тяжёлых |
| Игнор idle-in-transaction | таймаут + мониторинг |
| Изменения пачкой | по одному, с замером |

## Чек-лист

- [ ] `shared_buffers`/`effective_cache_size` под RAM.
- [ ] `work_mem` осторожно; тяжёлые запросы — `SET LOCAL`.
- [ ] Пулер спереди; пулы ≤ `max_connections`.
- [ ] Autovacuum включён; горячие таблицы тюнингованы.
- [ ] `ANALYZE` после bulk; statistics target поднят где нужно.
- [ ] Bloat/долгие транзакции/locks/wraparound мониторятся.
- [ ] `VACUUM FULL` только в окно.
