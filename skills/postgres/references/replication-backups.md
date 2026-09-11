# Репликация и бэкапы

## Физическая (streaming) vs логическая

|              | Physical/streaming       | Logical                         |
| ------------ | ------------------------ | ------------------------------- |
| Что копирует | весь кластер байт-в-байт | выбранные таблицы/схемы         |
| Версии       | одна и та же             | кросс-версии                    |
| Для чего     | HA, read-реплики         | выборочная репликация, апгрейды |
| `wal_level`  | `replica`                | `logical`                       |

## Настройка и мониторинг

```sql
-- primary: wal_level=replica; max_wal_senders=10; max_replication_slots=10
CREATE ROLE replicator WITH REPLICATION LOGIN PASSWORD '...';
SELECT * FROM pg_create_physical_replication_slot('replica_1');

-- базовая копия на standby
-- pg_basebackup -h primary -D $PGDATA -U replicator -P -v -R -X stream -S replica_1

-- lag (primary)
SELECT client_addr, state, sync_state, sent_lsn, replay_lsn,
       pg_wal_lsn_diff(sent_lsn, replay_lsn) AS lag_bytes
FROM pg_stat_replication;

-- слоты (следить, чтобы не забили диск)
SELECT slot_name, slot_type, active,
       pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn) AS retained_bytes
FROM pg_replication_slots;

-- lag (standby)
SELECT now() - pg_last_xact_replay_timestamp() AS replication_lag;
```

- **Replication slots** предотвращают удаление WAL — но неактивный слот может
  заполнить диск.
- Failover: `SELECT pg_promote()`; автоматизация — Patroni/pg_auto_failover.
- Синхронная репликация — `synchronous_commit=on`,
  `synchronous_standby_names`.

## Логическая репликация

```sql
-- publisher (wal_level=logical)
CREATE PUBLICATION my_pub FOR TABLE users, orders;
CREATE PUBLICATION active_users FOR TABLE users WHERE (active);  -- PG15 row filter

-- subscriber
CREATE SUBSCRIPTION my_sub
CONNECTION 'host=publisher dbname=mydb user=replicator'
PUBLICATION my_pub
WITH (copy_data = true, create_slot = true, enabled = true);

SELECT * FROM pg_stat_subscription;
```

## Бэкапы и PITR

- `pg_dump` — логический, по БД (малые/средние, отдельные таблицы).
- `pg_basebackup` — физический, весь кластер.
- Непрерывное архивирование WAL + **PITR** — восстановление на момент времени.

```bash
pg_basebackup -h localhost -U postgres -D /backup/base/$(date +%F) -Ft -z -P -X fetch
# recovery: recovery.signal + restore_command + recovery_target_time/xid/lsn
```

- Регулярно **тестировать восстановление**.
- Хранить бэкапы вне основного сервера; шифровать.

## Антипаттерны

| ❌                                     | ✅                      |
| -------------------------------------- | ----------------------- |
| Неактивный slot без мониторинга        | алерт по retained WAL   |
| Игнор replication lag                  | алерт (> 100MB / > 60s) |
| Нет теста рестора                      | регулярный restore-тест |
| Физическая репликация для кросс-версий | logical                 |
| Бэкапы только на том же сервере        | внешнее хранилище       |

## Чек-лист

- [ ] Тип репликации выбран осознанно.
- [ ] `wal_level`/слоты/senders настроены.
- [ ] Slots и lag мониторятся, есть алерты.
- [ ] Failover автоматизирован и протестирован.
- [ ] Бэкапы (base + WAL) настроены; рестор протестирован.
