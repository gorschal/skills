# Индексы

Индекс ускоряет чтение и замедляет запись. Создавать по паттерну запросов, а не
«на всякий случай».

## Типы

| Тип | Для чего |
|---|---|
| B-tree | равенство, диапазон, сортировка, join (по умолчанию) |
| GIN | JSONB, массивы, `tsvector`, `pg_trgm` |
| GiST | диапазоны, геометрия/PostGIS, KNN |
| BRIN | большие append-only, естественно упорядоченные (логи, time-series) |

## Правила проектирования

- **Селективность**: индекс окупается, если отсекает большинство строк.
  На `boolean` почти бесполезен → partial.
- **Составной**: порядок = равенство → диапазон → сортировка. Работает левый
  префикс; колонки после диапазона/пропуска — нет.
- **Partial**: узкое горячее подмножество.
- **Expression**: точное совпадение выражения с запросом.
- **Covering** (`INCLUDE`): Index Only Scan без похода в кучу.

```sql
-- составной + сортировка
CREATE INDEX idx_orders_user_date ON orders(user_id, created_at DESC);

-- partial
CREATE INDEX idx_orders_active ON orders(status, user_id)
  WHERE status IN ('pending','processing');

-- expression (должно совпадать с запросом: WHERE LOWER(email) = ...)
CREATE INDEX idx_users_lower_email ON users(LOWER(email));

-- covering
CREATE INDEX idx_orders_covering ON orders(user_id) INCLUDE (total, created_at);

-- GIN / GiST / BRIN
CREATE INDEX idx_docs_data  ON documents USING GIN(data);
CREATE INDEX idx_posts_tags ON posts USING GIN(tags);
CREATE INDEX idx_bookings_range ON bookings USING GIST(during);
CREATE INDEX idx_metrics_time_brin ON metrics USING BRIN(timestamp);

-- на живой таблице (НЕ в транзакции)
CREATE INDEX CONCURRENTLY idx_orders_user ON orders(user_id);
```

## Поиск проблемных индексов

```sql
-- seq scans по большим таблицам (кандидаты на индекс)
SELECT schemaname, tablename, seq_scan, seq_tup_read,
       seq_tup_read/NULLIF(seq_scan,0) AS avg_seq_tup_read
FROM pg_stat_user_tables
WHERE seq_scan > 0 AND seq_tup_read/NULLIF(seq_scan,0) > 10000
ORDER BY seq_tup_read DESC;

-- неиспользуемые индексы (смотреть после ~30 дней)
SELECT schemaname, tablename, indexname, idx_scan,
       pg_size_pretty(pg_relation_size(indexrelid)) AS size
FROM pg_stat_user_indexes
WHERE idx_scan = 0 AND indexrelname NOT LIKE '%pkey'
  AND indexrelname NOT LIKE 'pg_toast%'
ORDER BY pg_relation_size(indexrelid) DESC;
```

## Django / SQLAlchemy

- Django: `AddIndexConcurrently` (`django.contrib.postgres.operations`) +
  `atomic = False`; миграцию прогнать через `migration-safety`.
- SQLAlchemy: `Index("ix_...", "col", text("created_at DESC"))`, partial —
  `postgresql_where`.

## Антипаттерны

| ❌ | ✅ |
|---|---|
| Индексировать всё | по паттерну запросов |
| `(a)` + `(a,b)` | оставить `(a,b)` |
| `(created_at, user_id)` для `WHERE user_id=?` | equality-колонка первой |
| Полный индекс для 5% строк | partial |
| Индекс `email`, запрос `LOWER(email)` | expression-индекс |
| `CREATE INDEX` на живой таблице | `CONCURRENTLY` |
| Индексы без замера использования | проверить `EXPLAIN`/`idx_scan` |

## Чек-лист

- [ ] Колонки селективны; низкокардинальные — partial.
- [ ] Порядок колонок: равенство → диапазон → сортировка.
- [ ] `CONCURRENTLY` на проде.
- [ ] Подтверждено `EXPLAIN` до/после (Index/Index Only Scan).
- [ ] Проверены неиспользуемые/дублирующие индексы.
