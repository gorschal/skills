# JSONB

## Когда JSONB, когда колонки

| JSONB                            | Нормализованные колонки        |
| -------------------------------- | ------------------------------ |
| разреженные/разнородные атрибуты | частые фильтры/сортировки/join |
| внешние payload'ы, метаданные    | высокая кардинальность         |
| гибкая схема                     | строгая типизация, FK          |

Гибрид: JSONB для гибкой части, горячие ключи — в generated-колонки с B-tree.

## Операторы

```sql
data -> 'user' -> 'name'        -- JSONB
data ->> 'status'               -- text
data #> '{user,address,city}'   -- JSONB
data #>> '{user,address,city}'  -- text
data @> '{"status":"active"}'   -- containment (индексируемый)
data ? 'email'                  -- ключ существует
data ?| ARRAY['email','phone']  -- любой
data ?& ARRAY['email','phone']  -- все
```

`@>` — предпочтительный (индексируемый) оператор.

## Изменение

```sql
UPDATE documents SET data = data || '{"updated_at":"2024-01-01"}'::jsonb;
UPDATE documents SET data = data - 'temp_field';
UPDATE documents SET data = jsonb_set(data, '{user,email}', '"new@example.com"'::jsonb)
WHERE id = 123;
```

## Индексы

```sql
CREATE INDEX idx_docs_gin     ON documents USING GIN(data);
CREATE INDEX idx_docs_pathops ON documents USING GIN(data jsonb_path_ops);  -- только @>
CREATE INDEX idx_docs_status  ON documents((data->>'status'));
CREATE INDEX idx_docs_user_id ON documents(((data->'user'->>'id')::int));

-- generated-колонка + B-tree (лучший вариант для горячих скалярных полей)
ALTER TABLE documents
  ADD COLUMN status TEXT GENERATED ALWAYS AS (data->>'status') STORED;
CREATE INDEX idx_docs_status_col ON documents(status);
```

- `GIN(data)` — `@>`, `?`, `?|`, `?&`.
- `GIN(data jsonb_path_ops)` — меньше/быстрее, только `@>`.
- Expression B-tree — для равенства/диапазона по извлечённому значению.

## Запросы

```sql
SELECT data->>'status' AS status, COUNT(*), AVG((data->>'score')::float)
FROM documents GROUP BY data->>'status';

SELECT jsonb_agg(data->'user') FROM documents WHERE data @> '{"status":"active"}';

-- path-запросы (PG12+)
SELECT jsonb_path_query(data, '$.items[*] ? (@.price > 100)') FROM documents;

-- валидация схемы (PG15)
ALTER TABLE documents ADD CONSTRAINT check_data_schema CHECK (
  jsonb_typeof(data) = 'object' AND data ? 'id' AND data ? 'status'
  AND data->>'status' IN ('active','pending','archived')
);
```

## Ограничения

- `jsonb` (binary, индексируемый), не `json`.
- Не хранить огромные массивы (10k+) — отдельная таблица.
- Не оставлять часто обновляемые поля в JSONB.
- Следить за типами: число vs JSON-строка ломает индекс/сравнение.

## Антипаттерны

| ❌                                  | ✅                         |
| ----------------------------------- | -------------------------- |
| `json` вместо `jsonb`               | `jsonb`                    |
| `->>` сравнение вместо `@>`         | `@>` + GIN                 |
| GIN на всё, когда нужен только `@>` | `jsonb_path_ops`           |
| Горячее поле только в JSONB         | generated-колонка + B-tree |
| Большие массивы в JSONB             | отдельная таблица          |
| Разные типы в поле                  | единый тип                 |

## Чек-лист

- [ ] `jsonb`, не `json`.
- [ ] `@>` для containment + соответствующий GIN.
- [ ] `jsonb_path_ops`, если нужен только `@>`.
- [ ] Горячие скаляры вынесены в (generated) колонки с B-tree.
- [ ] Нет больших массивов и частых апдейтов JSONB.
- [ ] (PG15) `CHECK` валидирует форму.
