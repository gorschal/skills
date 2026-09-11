# Диагностика PostgreSQL

Правило: **сначала измерить, потом лечить**. Read-only диагностика безопасна
всегда; изменения — отдельный явный шаг.

## Контекст

```sql
SELECT version();
SELECT pg_database_size(current_database());
SELECT relname, n_live_tup, n_dead_tup FROM pg_stat_user_tables ORDER BY n_dead_tup DESC LIMIT 20;
```

Уточнить: ORM (Django/SQLAlchemy), доступ к psql на проде, симптом (один запрос /
страница / «всё» / пики по времени).

## Поиск виновника

```sql
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;   -- требует shared_preload_libraries + рестарт

SELECT query, calls, total_exec_time, mean_exec_time, max_exec_time, rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;

SELECT pg_stat_statements_reset();
```

Без расширения: `log_min_duration_statement = 500` (мс) → медленные запросы в лог.

На dev: `django-debug-toolbar`/`silk` — какие SQL порождает страница (N+1 — это
уровень ORM, навыки `django`/`fastapi`).

## Чтение плана

```sql
EXPLAIN (ANALYZE, BUFFERS, VERBOSE) <query>;
```

Сравнивать `rows` (estimate) и `actual rows`; `Buffers: shared hit` vs `read`.
Node preference: Index Only Scan > Index Scan > Bitmap Index Scan > Seq Scan.

| Симптом                                      | Причина               | Лечение                                          |
| -------------------------------------------- | --------------------- | ------------------------------------------------ |
| `Seq Scan` на большой таблице + узкий фильтр | нет индекса           | индекс по фильтру                                |
| estimate `rows=1000000`, actual `rows=10`    | устаревшая статистика | `ANALYZE tbl`, поднять statistics target         |
| `Sort Method: external merge Disk`           | сортировка не влезла  | `work_mem`, индекс под `ORDER BY`                |
| `Nested Loop` с большим внешним набором      | плохой join-план      | индекс по join-колонке, статистика               |
| `Rows Removed by Filter: <огромное>`         | индекс не подходит    | составной/partial индекс                         |
| высокие `Buffers: read` при повторе          | нет в кеше/bloat      | `shared_buffers`, `effective_cache_size`, vacuum |

`Seq Scan` нормален для маленьких таблиц и чтения большей части таблицы.

## Методика

1. Baseline: сохранить план и время.
2. Одно изменение.
3. Повторный `EXPLAIN (ANALYZE, BUFFERS)` тем же запросом.
4. Сравнить cost и wall-clock; зафиксировать.

## Уровни риска

| Действие                               | Риск                         |
| -------------------------------------- | ---------------------------- |
| `EXPLAIN`, `pg_stat_*`                 | безопасно всегда             |
| `CREATE INDEX CONCURRENTLY`, `ANALYZE` | безопасно, но I/O — не в пик |
| `postgresql.conf`, pgbouncer           | окно/перезагрузка            |
| `VACUUM FULL`, schema change, failover | окно + план отката           |

## Чек-лист

- [ ] Виновник найден измерением, не догадкой.
- [ ] План прочитан: estimate ≈ actual.
- [ ] Нет лишних `Seq Scan` по большим таблицам.
- [ ] Изменения по одному, с замером до/после.
- [ ] Изменения на проде — явным шагом с откатом.
