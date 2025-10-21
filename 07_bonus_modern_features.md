## 📘 `07_bonus_modern_features.md`

# Современные возможности PostgreSQL и PL/pgSQL * (расширенные приёмы)

---

## 🧭 Оглавление

* [1. Новые фичи PostgreSQL 15–17*](#1-новые-фичи-postgresql-15–17)
* [2. Расширенные возможности JSONB и агрегатов*](#2-расширенные-возможности-jsonb-и-агрегатов)
* [3. Использование RETURNING, CTE и UPSERT в функциях*](#3-использование-returning-cte-и-upsert-в-функциях)
* [4. Расширения для аудита, планирования и аналитики*](#4-расширения-для-аудита-планирования-и-аналитики)
* [5. Параллелизм, фоновые задачи и планировщики*](#5-параллелизм-фоновые-задачи-и-планировщики)
* [6. Примеры задач со звёздочкой](#6-примеры-задач-со-звёздочкой)
* [7. Трюки и нестандартные решения*](#7-трюки-и-нестандартные-решения)
* [8. Рекомендованные расширения и утилиты](#8-рекомендованные-расширения-и-утилиты)

---

## 1. Новые фичи PostgreSQL 15–17*

### ⭐ 1.1 `MERGE` — теперь официально в PostgreSQL 15+

Комбинированная операция “insert or update” без ручных условий.

```sql
MERGE INTO employees AS e
USING updates AS u ON e.id = u.id
WHEN MATCHED THEN
    UPDATE SET salary = u.salary
WHEN NOT MATCHED THEN
    INSERT (id, name, salary) VALUES (u.id, u.name, u.salary);
```

💡 *Заменяет громоздкие конструкции с `INSERT ... ON CONFLICT ... DO UPDATE`.*

---

### ⭐ 1.2 `GENERATED ALWAYS AS IDENTITY` (PostgreSQL 13+)

Новый стандарт вместо `SERIAL`:

```sql
CREATE TABLE users (
    id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name TEXT
);
```

🟢 Работает корректно с дампами, резервными копиями и управлением последовательностей.

---

### ⭐ 1.3 `SEARCH` и `CYCLE` для рекурсивных запросов (PostgreSQL 14+)

Теперь можно писать красивые древовидные запросы:

```sql
WITH RECURSIVE org_chart(id, name, manager_id) AS (
  SELECT id, name, manager_id FROM employees WHERE manager_id IS NULL
  UNION ALL
  SELECT e.id, e.name, e.manager_id
  FROM employees e JOIN org_chart o ON e.manager_id = o.id
)
SEARCH DEPTH FIRST BY name SET order_col;
```

---

### ⭐ 1.4 `pg_stat_io` (PostgreSQL 16+)

Новая системная таблица для анализа I/O — можно следить за дисковыми нагрузками.

```sql
SELECT * FROM pg_stat_io WHERE backend_type = 'user backend';
```

---

### ⭐ 1.5 `parallel_commit` и `logical replication slots` (PostgreSQL 17*)

* Поддержка параллельных коммитов при репликации;
* Более точное управление слотом логической репликации (важно при CDC, Debezium, Kafka).

---

## 2. Расширенные возможности JSONB и агрегатов*

### ⭐ 2.1 `jsonb_set()` — обновление значения по ключу

```sql
UPDATE users
SET data = jsonb_set(data, '{email}', '"new@mail.ru"');
```

---

### ⭐ 2.2 `jsonb_path_query` и JSONPath

Мощный способ фильтрации данных внутри JSON:

```sql
SELECT jsonb_path_query(data, '$.users[*] ? (@.active == true)');
```

---

### ⭐ 2.3 Агрегация JSONB

```sql
SELECT jsonb_agg(jsonb_build_object('name', name, 'salary', salary))
FROM employees;
```

💡 Используется при сборке API-ответов или при работе с REST/GraphQL слоями.

---

## 3. Использование RETURNING, CTE и UPSERT в функциях*

### ⭐ 3.1 Возврат вставленных строк через RETURNING

```sql
CREATE OR REPLACE FUNCTION add_user(username TEXT, email TEXT)
RETURNS TABLE(id INT, username TEXT) AS $$
BEGIN
    RETURN QUERY
    INSERT INTO users (username, email)
    VALUES (username, email)
    RETURNING id, username;
END;
$$ LANGUAGE plpgsql;
```

---

### ⭐ 3.2 Сложные операции через CTE (`WITH`)

```sql
CREATE OR REPLACE FUNCTION update_or_insert_order(p_id INT, p_qty INT)
RETURNS VOID AS $$
BEGIN
    WITH upsert AS (
        UPDATE orders SET quantity = p_qty WHERE id = p_id RETURNING *
    )
    INSERT INTO orders (id, quantity)
    SELECT p_id, p_qty
    WHERE NOT EXISTS (SELECT 1 FROM upsert);
END;
$$ LANGUAGE plpgsql;
```

---

### ⭐ 3.3 `ON CONFLICT DO UPDATE`

```sql
INSERT INTO employees (id, name, salary)
VALUES (1, 'Alex', 100000)
ON CONFLICT (id) DO UPDATE
SET salary = EXCLUDED.salary;
```

---

## 4. Расширения для аудита, планирования и аналитики*

| Расширение           | Назначение                                                   |
| -------------------- | ------------------------------------------------------------ |
| `pg_stat_statements` | профайлинг SQL-запросов                                      |
| `pg_cron`            | запуск SQL-планировщика внутри PostgreSQL                    |
| `pgvector`           | хранение и поиск векторных эмбеддингов (AI, LLM)             |
| `timescaledb`        | расширение для временных рядов                               |
| `pg_partman`         | автоматическое управление партициями                         |
| `hypopg`             | создание “виртуальных” индексов для тестирования оптимизации |

---

### ⭐ Пример использования `pg_cron`

```sql
SELECT cron.schedule('cleanup_old_audit', '0 2 * * *', $$
    DELETE FROM audit_log WHERE changed_at < NOW() - INTERVAL '90 days';
$$);
```

---

### ⭐ Пример использования `pgvector`

```sql
CREATE EXTENSION IF NOT EXISTS vector;

CREATE TABLE documents (
    id SERIAL,
    content TEXT,
    embedding VECTOR(1536)
);

-- Поиск ближайших по косинусному расстоянию
SELECT id, embedding <=> '[0.12, 0.98, ...]' AS distance
FROM documents
ORDER BY distance
LIMIT 5;
```

---

## 5. Параллелизм, фоновые задачи и планировщики*

### ⭐ 5.1 Параллельные запросы

```sql
SET max_parallel_workers_per_gather = 4;
SELECT SUM(amount) FROM transactions;
```

---

### ⭐ 5.2 LISTEN / NOTIFY — нотификации между сессиями

```sql
-- Сессия 1:
LISTEN data_updated;

-- Сессия 2:
NOTIFY data_updated, 'Table users was updated';
```

💡 Отлично подходит для триггеров, которые уведомляют внешние сервисы (Kafka, Python, Go).

---

### ⭐ 5.3 Планирование через pg_cron и системные функции

```sql
SELECT clock_timestamp(), statement_timestamp(), transaction_timestamp();
```

---

## 6. Примеры задач со звёздочкой

---

### 🧩 Задача 1* — динамическая генерация SQL внутри функции

```sql
CREATE OR REPLACE FUNCTION dynamic_select(table_name TEXT)
RETURNS SETOF RECORD AS $$
BEGIN
    RETURN QUERY EXECUTE format('SELECT * FROM %I LIMIT 5', table_name);
END;
$$ LANGUAGE plpgsql;
```

**Проверка:**

```sql
SELECT * FROM dynamic_select('users') AS t(id INT, username TEXT, email TEXT);
```

---

### 🧩 Задача 2* — аудит только изменённых колонок

```sql
CREATE OR REPLACE FUNCTION audit_changed_columns()
RETURNS TRIGGER AS $$
DECLARE
    diff JSONB;
BEGIN
    diff := (
        SELECT jsonb_object_agg(key, value)
        FROM jsonb_each(to_jsonb(NEW))
        WHERE (to_jsonb(OLD)->key) IS DISTINCT FROM value
    );

    IF diff IS NOT NULL THEN
        INSERT INTO audit_log (table_name, operation, new_data)
        VALUES (TG_TABLE_NAME, TG_OP, diff);
    END IF;

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

---

### 🧩 Задача 3* — автоархивация старых данных

```sql
CREATE OR REPLACE FUNCTION archive_old_records()
RETURNS VOID AS $$
BEGIN
    INSERT INTO archive_table SELECT * FROM orders WHERE created_at < NOW() - INTERVAL '1 year';
    DELETE FROM orders WHERE created_at < NOW() - INTERVAL '1 year';
END;
$$ LANGUAGE plpgsql;
```

Можно запланировать через `pg_cron`.

---

## 7. Трюки и нестандартные решения*

* Используй `to_regclass('table_name')` — безопасная проверка существования таблицы.
* `CREATE FUNCTION ... RETURNS TRIGGER SECURITY DEFINER` — выполнять триггер от имени владельца (часто нужно при RLS).
* `EXPLAIN (ANALYZE, BUFFERS)` — анализ производительности прямо в DBeaver.
* `pg_partman` + `pg_cron` = автоматическое партиционирование по дате.
* Для JSONB-диффов можно использовать `jsonb_diff()` (через расширение `pg_jsonschema`).

---

## 8. Рекомендованные расширения и утилиты

| Расширение             | Назначение                            |
| ---------------------- | ------------------------------------- |
| **pg_stat_statements** | анализ медленных запросов             |
| **pg_cron**            | встроенный cron для PostgreSQL        |
| **pg_partman**         | управление партициями                 |
| **pgvector**           | хранение эмбеддингов                  |
| **pgagent**            | задачи и фоновые скрипты              |
| **timescaledb**        | временные ряды и аналитика            |
| **plpython3u**         | запуск Python внутри PostgreSQL       |
| **pgaudit**            | аудит DDL/DML операций                |
| **hypopg**             | “виртуальные индексы” для оптимизации |

---

✅ После этой главы ты знаешь:

* новейшие возможности PostgreSQL 15–17;
* продвинутые конструкции PL/pgSQL и JSONB;
* умеешь создавать автоматизированный аудит, фоновые задачи и расширения;
* знаешь приёмы, которые отличают senior-инженера от middle-разработчика.

---
