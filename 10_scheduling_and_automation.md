# 10_scheduling_and_automation.md

## Планирование и автоматизация задач в PostgreSQL и ClickHouse

---

## 📑 Оглавление

- [10\_scheduling\_and\_automation.md](#10_scheduling_and_automationmd)
  - [Планирование и автоматизация задач в PostgreSQL и ClickHouse](#планирование-и-автоматизация-задач-в-postgresql-и-clickhouse)
  - [📑 Оглавление](#-оглавление)
    - [🧭 Введение](#-введение)
    - [🗺️ Общая схема (ASCII)](#️-общая-схема-ascii)
  - [1. Планирование задач в PostgreSQL](#1-планирование-задач-в-postgresql)
    - [1.1. pg\_cron](#11-pg_cron)
    - [1.2. pgAgent](#12-pgagent)
    - [1.3. Сравнение](#13-сравнение)
  - [2. Автоматизация в ClickHouse](#2-автоматизация-в-clickhouse)
    - [2.1. SYSTEM-команды](#21-system-команды)
    - [2.2. Materialized Views](#22-materialized-views)
    - [2.3. Очистка и оптимизация](#23-очистка-и-оптимизация)
  - [3. Интеграция PostgreSQL → ClickHouse (ETL)](#3-интеграция-postgresql--clickhouse-etl)
    - [3.1. SQL-процедура в PostgreSQL](#31-sql-процедура-в-postgresql)
    - [3.2. Bash + cron](#32-bash--cron)
  - [4. Автоматизация бэкапов и отчётов](#4-автоматизация-бэкапов-и-отчётов)
    - [4.1. Бэкап PostgreSQL](#41-бэкап-postgresql)
    - [4.2. Бэкап ClickHouse](#42-бэкап-clickhouse)
    - [4.3. Отчёты CSV](#43-отчёты-csv)
  - [5. Практика](#5-практика)
  - [6. Контрольные и интервью-вопросы](#6-контрольные-и-интервью-вопросы)
  - [🧩 Итог](#-итог)

---

### 🧭 Введение

Автоматизация позволяет устранять рутинные задачи — экспорты, отчёты, очистку логов и резервное копирование.

В PostgreSQL используются **pg_cron** и **pgAgent**, в ClickHouse — `SYSTEM`-команды, `MATERIALIZED VIEW` и cron-скрипты.

---

### 🗺️ Общая схема (ASCII)

```
[Пользователь] → PostgreSQL → pg_cron / Триггеры → ClickHouse → Bash + cron (бэкапы, отчёты)
```

---

## 1. Планирование задач в PostgreSQL

### 1.1. pg_cron

```sql
CREATE EXTENSION IF NOT EXISTS pg_cron;
SELECT cron.schedule('0 2 * * *', $$CALL export_audit_to_csv();$$);
```

Процедура:

```sql
CREATE OR REPLACE PROCEDURE export_audit_to_csv()
LANGUAGE plpgsql AS $$
BEGIN
 COPY (SELECT * FROM users_audit WHERE changed_at>NOW()-INTERVAL '1 day')
 TO '/var/lib/postgresql/exports/users_audit.csv' WITH CSV HEADER;
END; $$;
```

### 1.2. pgAgent

```
apt install pgagent
pgagent hostaddr=127.0.0.1 dbname=example_db user=user
```

### 1.3. Сравнение

| Параметр | pg_cron        | pgAgent              |
| -------- | -------------- | -------------------- |
| Простота | Легко          | Сложнее              |
| Формат   | Cron-строка    | GUI / SQL            |
| Подходит | Простые задачи | Многошаговые цепочки |

---

## 2. Автоматизация в ClickHouse

### 2.1. SYSTEM-команды

```sql
SYSTEM FLUSH LOGS;
SYSTEM MERGES;
SYSTEM OPTIMIZE TABLE user_stats FINAL;
```

### 2.2. Materialized Views

```sql
CREATE MATERIALIZED VIEW mv_daily_stats
ENGINE = SummingMergeTree()
ORDER BY (date)
AS
SELECT toDate(created_at) AS date, count() AS users_created
FROM users_audit_ch GROUP BY date;
```

### 2.3. Очистка и оптимизация

```sql
ALTER TABLE users_audit_ch DELETE WHERE changed_at < now() - INTERVAL 30 DAY;
OPTIMIZE TABLE users_audit_ch FINAL;
```

---

## 3. Интеграция PostgreSQL → ClickHouse (ETL)

### 3.1. SQL-процедура в PostgreSQL

```sql
CREATE OR REPLACE PROCEDURE export_audit_to_ch()
LANGUAGE plpgsql AS $$
BEGIN
 COPY (SELECT * FROM users_audit WHERE changed_at>NOW()-INTERVAL '1 day')
 TO PROGRAM 'clickhouse-client --query="INSERT INTO users_audit_ch FORMAT CSV"'
 WITH CSV;
END; $$;
SELECT cron.schedule('0 1 * * *','CALL export_audit_to_ch();');
```

### 3.2. Bash + cron

```bash
#!/bin/bash
DATE=$(date +"%Y-%m-%d")
PG_CMD="COPY (SELECT * FROM users_audit WHERE changed_at > NOW() - INTERVAL '1 day') TO STDOUT WITH CSV HEADER"
psql -U user -d example_db -c "$PG_CMD" | clickhouse-client --query="INSERT INTO users_audit_ch FORMAT CSV"
echo "[$DATE] Export complete" >> /var/log/pg_ch_sync.log
```

Cron: `0 1 * * * /scripts/pg_to_ch_sync.sh`

---

## 4. Автоматизация бэкапов и отчётов

### 4.1. Бэкап PostgreSQL

```bash
#!/bin/bash
DATE=$(date +"%Y-%m-%d_%H-%M")
pg_dump -U user example_db > /backups/postgres_$DATE.sql
find /backups -type f -mtime +7 -delete
```

### 4.2. Бэкап ClickHouse

```bash
#!/bin/bash
DATE=$(date +"%Y-%m-%d_%H-%M")
clickhouse-backup create daily_$DATE
clickhouse-backup delete local --older-than 7d
```

### 4.3. Отчёты CSV

```bash
#!/bin/bash
psql -U user -d example_db -c "\\copy (SELECT role,COUNT(*) FROM users GROUP BY role) TO '/exports/user_roles.csv' WITH CSV HEADER"
```

Cron: `0 6 * * 1 /scripts/weekly_report.sh`

---

## 5. Практика

* Cron-задача очистки `temp_logs` от старше 3 дней.
* Bash-бэкап ClickHouse в 04:00.
* Materialized View по ролям.
* Процедура `sync_pg_to_ch()` ежедневная.
* Cron-отчёт каждую субботу в 23:59.

Пример:

```sql
CREATE OR REPLACE PROCEDURE clear_temp_logs()
LANGUAGE plpgsql AS $$
BEGIN
 DELETE FROM temp_logs WHERE created_at < NOW()-INTERVAL '3 days';
END; $$;
SELECT cron.schedule('0 * * * *','CALL clear_temp_logs();');
```

---

## 6. Контрольные и интервью-вопросы

1. Разница pg_cron и pgAgent?
2. Как запустить внешние команды через PostgreSQL?
3. Что делает COPY TO PROGRAM?
4. Как обновляется Materialized View?
5. Что делает SYSTEM OPTIMIZE TABLE?
6. Как автоматизировать ежедневный бэкап?
7. Можно ли cron внутри Docker?

---

## 🧩 Итог

* PostgreSQL: pg_cron / pgAgent.
* ClickHouse: SYSTEM / MV / cron.
* PostgreSQL → ClickHouse = ETL.
* Bash cron дополняет резервные копии и отчёты.

---

**Далее:** → [11_data_streams_and_integrations.md](11_data_streams_and_integrations.md)
