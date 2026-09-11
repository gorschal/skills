# Postgres: операции, блокировки, безопасные альтернативы

Источник проблем — DDL берёт `ACCESS EXCLUSIVE` lock (блокирует **всё**, включая
чтение) и/или переписывает таблицу (table rewrite). Плюс **очередь блокировок**:
даже мгновенный `ACCESS EXCLUSIVE` ждёт текущие запросы, а пока ждёт — блокирует
новые. Поэтому «быстрая» операция на нагруженной таблице может остановить её на
время самого долгого активного запроса.

Версии важны: часть операций безопасна с PG 11/12+.

## Таблица операций

| Операция | Риск | Почему | Безопасно |
|---|---|---|---|
| `ADD COLUMN` без default | LOW | метаданные, мгновенно | да |
| `ADD COLUMN` с **константным** default | LOW (PG11+) | default в каталоге, без rewrite | да на PG11+ |
| `ADD COLUMN` с **volatile** default (`now()`, `uuid_generate_v4()`) | CRITICAL | rewrite под `ACCESS EXCLUSIVE` | nullable → backfill → default отдельно |
| `ADD COLUMN ... NOT NULL` без default на непустой | CRITICAL | ошибка/rewrite | nullable → backfill → `NOT NULL` |
| `ALTER COLUMN SET NOT NULL` | HIGH | полный скан под `ACCESS EXCLUSIVE` (PG≤11) | PG12+: `CHECK (col IS NOT NULL) NOT VALID` → `VALIDATE` → `SET NOT NULL` |
| `CREATE INDEX` без `CONCURRENTLY` | HIGH | `SHARE` lock — блокирует запись на всё построение | `CREATE INDEX CONCURRENTLY` (вне транзакции) |
| `ALTER COLUMN TYPE` | CRITICAL | почти всегда rewrite + `ACCESS EXCLUSIVE` | новая колонка → backfill → переключение |
| `ADD FOREIGN KEY` | HIGH | `SHARE ROW EXCLUSIVE` + скан на валидацию | `NOT VALID` → `VALIDATE` |
| `ADD UNIQUE` | HIGH | строит unique-индекс под локом | `CREATE UNIQUE INDEX CONCURRENTLY` → `USING INDEX` |
| `ADD CHECK` | MEDIUM | скан на валидацию под локом | `NOT VALID` → `VALIDATE` |
| `DROP COLUMN` | HIGH | быстро, но теряет данные и ломает старый код | только contract-фаза |
| `RENAME COLUMN`/`TABLE` | HIGH | мгновенно ломает старый код | не для zero-downtime; expand/contract |
| `DROP TABLE`/`TRUNCATE` | CRITICAL | потеря данных; `TRUNCATE` берёт `ACCESS EXCLUSIVE` | после подтверждения + бэкап |
| Backfill всей таблицы | CRITICAL | длинные локи, WAL, лаг реплик | батчами по PK с commit |

## CREATE INDEX CONCURRENTLY

- Не внутри транзакции: в Django — `atomic = False`.
- Дольше обычного, два прохода.
- При сбое оставляет **invalid index** — найти (`pg_index.indisvalid`) и
  `DROP INDEX CONCURRENTLY` перед повтором.

## NOT VALID → VALIDATE

```sql
-- Фаза 1: быстро; существующие строки не проверяются
ALTER TABLE orders ADD CONSTRAINT orders_user_fk
    FOREIGN KEY (user_id) REFERENCES users(id) NOT VALID;

-- Фаза 2: сканирует существующие строки под SHARE UPDATE EXCLUSIVE (DML не блокируется)
ALTER TABLE orders VALIDATE CONSTRAINT orders_user_fk;
```

## Backfill больших таблиц

Никогда не обновлять всю таблицу одной транзакцией. Безопасно — батчами по PK,
с commit между батчами, вне транзакции миграции, идемпотентно:

```sql
-- повторять, пока затронуто > 0; между итерациями — commit
UPDATE big_table SET new_col = old_col
WHERE id BETWEEN :lo AND :hi AND new_col IS NULL;
```

Большой backfill лучше вынести в management-команду/Django Task, чтобы деплой не
висел и его можно было ставить на паузу.

## Таймауты перед DDL

```sql
SET lock_timeout = '3s';        -- не ждать лок дольше 3с (упасть, не копить очередь)
SET statement_timeout = '0';    -- дать самой операции отработать (или лимит)
```

При срабатывании `lock_timeout` миграция падает чисто — повторить в момент меньшей
нагрузки. Это лучше зависшей миграции, заблокировавшей трафик.

## Чек-лист

- [ ] Тяжёлые операции — `CONCURRENTLY`/батчи/`NOT VALID`→`VALIDATE`.
- [ ] Перед DDL выставлен `lock_timeout`.
- [ ] Backfill — батчами по PK, идемпотентно, вне транзакции.
- [ ] Оценено на объёме прода.
- [ ] После сбоя проверены invalid-индексы.
