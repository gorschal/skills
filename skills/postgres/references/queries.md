# Оптимизация запросов

## Общие правила

- Всегда читать `EXPLAIN (ANALYZE, BUFFERS)`.
- `Seq Scan` приемлем для маленьких таблиц; проблема — на больших с узким фильтром.
- Индексировать колонки join и фильтров; держать статистику свежей (`ANALYZE`).
- **N+1** — уровень ORM: `select_related`/`prefetch_related`,
  `joinedload`/`selectinload`. Здесь — когда SQL уже правильный, но медленный.

## Плохие паттерны

| Паттерн | Проблема | Решение |
|---|---|---|
| `SELECT *` | лишний I/O | только нужные колонки |
| `OR` по колонкам | блокирует индекс | `UNION`/отдельные запросы |
| `LIKE '%term%'` | полный скан | `pg_trgm` GIN или FTS |
| `WHERE DATE(col)=...` | функция блокирует индекс | диапазон |
| `IN` > ~100 элементов | неэффективно | temp-таблица/JOIN |
| неявный каст (`id = '123'`) | блокирует индекс | совпадение типов |
| большой `OFFSET` | сканирует и отбрасывает | keyset-пагинация |

```sql
-- функция на колонке убивает индекс:
-- BAD:  WHERE DATE(created_at) = '2024-01-01'
-- GOOD: WHERE created_at >= '2024-01-01' AND created_at < '2024-01-02'

-- EXISTS вместо IN (короткое замыкание)
SELECT u.* FROM users u
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.user_id = u.id AND o.total > 1000);
```

## Keyset-пагинация

```sql
SELECT * FROM products
WHERE (created_at, id) < ('2024-01-01 12:00:00', 12345)
ORDER BY created_at DESC, id DESC
LIMIT 20;

-- поддерживающий индекс
CREATE INDEX idx_products_pagination ON products (created_at DESC, id DESC);
```

`OFFSET N` читает и отбрасывает N строк — на больших N деградирует.

## CTE

```sql
-- принудительная материализация (переиспользование)
WITH expensive AS MATERIALIZED (
  SELECT user_id, SUM(total) v FROM orders GROUP BY user_id
)
SELECT * FROM expensive WHERE v > 10000;

-- принудительный inline
WITH recent AS NOT MATERIALIZED (
  SELECT id FROM users WHERE created_at > now() - interval '7 days'
)
SELECT * FROM recent;
```

С PG12 не-рекурсивные CTE с одним использованием инлайнятся по умолчанию.

## Агрегации и окна

```sql
-- окно вместо коррелированных подзапросов
SELECT id, total,
       MAX(total) OVER (PARTITION BY user_id) AS max_total
FROM orders;

-- приблизительный count вместо COUNT(*)
SELECT reltuples::bigint FROM pg_class WHERE relname = 'orders';

-- материализованный счётчик для отчётов
CREATE MATERIALIZED VIEW order_counts AS
SELECT status, COUNT(*) FROM orders GROUP BY status;
CREATE UNIQUE INDEX ON order_counts(status);
REFRESH MATERIALIZED VIEW CONCURRENTLY order_counts;
```

## Joins

- Nested Loop — маленький внешний + индексированный внутренний.
- Hash Join — большие неотсортированные наборы.
- Merge Join — отсортированные/большие.
- Убедиться, что колонки join индексированы и статистика свежая.

## Антипаттерны

| ❌ | ✅ |
|---|---|
| `SELECT *` | нужные колонки |
| Функция на индексируемой колонке | диапазон |
| `LIKE '%term%'` без индекса | `pg_trgm`/FTS |
| `OR` по колонкам | `UNION` |
| `IN` с сотнями значений | JOIN/temp-таблица |
| `OFFSET 100000` | keyset |
| Коррелированные подзапросы | окна/join |
| N+1 в ORM | eager loading |

## Чек-лист

- [ ] `EXPLAIN (ANALYZE, BUFFERS)` до и после.
- [ ] Нет `Seq Scan` по большим таблицам без причины.
- [ ] Колонки join/фильтра индексированы; статистика свежая.
- [ ] Пагинация keyset.
- [ ] CTE материализуются/инлайнятся осознанно.
- [ ] N+1 устранён на уровне ORM.
