# Расширения PostgreSQL

## pg_stat_statements (обязательно)

```sql
-- shared_preload_libraries = 'pg_stat_statements'  (требует рестарта)
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

SELECT query, calls, total_exec_time, mean_exec_time, max_exec_time, rows
FROM pg_stat_statements ORDER BY mean_exec_time DESC LIMIT 10;

SELECT pg_stat_statements_reset();
```

Основа поиска медленных запросов.

## pg_trgm (fuzzy / ILIKE)

```sql
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE INDEX idx_users_email_trgm ON users USING GIN (email gin_trgm_ops);

SELECT * FROM users WHERE email ILIKE '%john%';
SELECT * FROM users WHERE email % 'jon@example.com';   -- similarity
SET pg_trgm.similarity_threshold = 0.5;
```

Для `LIKE '%term%'`, которых не покрывает B-tree.

## UUID и crypto

```sql
-- PG13+: встроенный gen_random_uuid(), расширение не нужно
SELECT gen_random_uuid();

CREATE EXTENSION IF NOT EXISTS pgcrypto;
SELECT gen_random_bytes(32);
SELECT digest('data','sha256');
SELECT pgp_sym_encrypt('secret','key');
```

В Django пароли хешируются приложением (PBKDF2/Argon2), не `pgcrypto`.

## citext (регистронезависимый текст)

```sql
CREATE EXTENSION IF NOT EXISTS citext;
-- тип citext; альтернатива expression-индексу на LOWER(email)
```

## btree_gin

```sql
CREATE EXTENSION IF NOT EXISTS btree_gin;
-- скалярные колонки участвуют в GIN: equality + array/JSONB containment
CREATE INDEX idx_posts_status_tags ON posts USING GIN (status, tags);
```

## PostGIS (spatial)

```sql
CREATE EXTENSION IF NOT EXISTS postgis;
CREATE TABLE locations (id serial PRIMARY KEY, name text, geom geometry(Point,4326));
CREATE INDEX idx_locations_geom ON locations USING GIST (geom);

SELECT * FROM locations
WHERE ST_DWithin(geom::geography,
                 ST_SetSRID(ST_MakePoint(-74.006,40.7128),4326)::geography, 1000);
```

## Антипаттерны

| ❌                                        | ✅                       |
| ----------------------------------------- | ------------------------ |
| Нет `pg_stat_statements`                  | включить (нужен рестарт) |
| `LIKE '%term%'` без индекса               | `pg_trgm` GIN            |
| `uuid-ossp` на PG13+                      | `gen_random_uuid()`      |
| Хешировать пароли в БД                    | хеш в приложении         |
| `citext` там, где нужен expression-индекс | выбрать осознанно        |
| PostGIS без GiST                          | GiST по геометрии        |

## Чек-лист

- [ ] `pg_stat_statements` включён и предзагружен.
- [ ] `pg_trgm` GIN для `ILIKE '%...%'`/fuzzy.
- [ ] `gen_random_uuid()` вместо `uuid-ossp` (PG13+).
- [ ] Крипто — в приложении.
- [ ] Spatial-предикаты индексированы GiST.
