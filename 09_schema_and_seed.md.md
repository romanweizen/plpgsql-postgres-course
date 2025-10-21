## 📘 `09_schema_and_seed.md`

# Схема БД и наполнение тестовыми данными для всех примеров курса

---

## 🧭 Оглавление

* [0. Общие рекомендации по запуску](#0-общие-рекомендации-по-запуску)
* [1. Базовая подготовка: схема, расширения, application_name](#1-базовая-подготовка-схема-расширения-application_name)
* [2. Таблица employees (используется в Частях 1–2, 5)](#2-таблица-employees-используется-в-частях-1–2-5)
* [3. Таблица users (используется в Частях 2–5)](#3-таблица-users-используется-в-частях-2–5)
* [4. История users_history (лог email; Часть 3)](#4-история-users_history-лог-email-часть-3)
* [5. Универсальный JSONB-аудит audit_log (Части 3–5)](#5-универсальный-jsonb-аудит-audit_log-части-3–5)
* [6. Таблицы для бизнес-кейса заказов: products, orders (Часть 5)](#6-таблицы-для-бизнес-кейса-заказов-products-orders-часть-5)
* [7. deleted_users (лог удаления; Часть 3 — самостоятелька)](#7-deleted_users-лог-удаления-часть-3--самостоятелька)
* [8. events (JSON/лог-событий — примеры JSONB/ClickHouse)](#8-events-jsonлог-событий--примеры-jsonbclickhouse)
* [9. Вспомогательные таблицы для примеров архивации](#9-вспомогательные-таблицы-для-примеров-архивации)
* [10. Быстрые проверки и полезные индексы](#10-быстрые-проверки-и-полезные-индексы)

---

## 0. Общие рекомендации по запуску

* Выполняй скрипт **по порядку секций**.
* Если уже пробовал предыдущие части — начни с очистки/пересоздания схемы в разделе 1.
* Все `id` — через современный синтаксис `GENERATED ALWAYS AS IDENTITY`.
* Даты и числа подобраны так, чтобы примеры из курса давали наглядные результаты.

---

## 1. Базовая подготовка: схема, расширения, application_name

```sql
-- (опционально) создадим отдельную схему курса
CREATE SCHEMA IF NOT EXISTS course;
SET search_path TO course, public;

-- полезные расширения (не обязательны для DDL, но пригодятся в практике)
-- CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
-- CREATE EXTENSION IF NOT EXISTS pgaudit;

-- для логов приложений в аудите
SET application_name = 'course_setup';
```

---

## 2. Таблица employees (используется в Частях 1–2, 5)

**Что демонстрируем:** SELECT INTO, RETURN QUERY, UPDATE, FOUND/ROW_COUNT, агрегаты.

```sql
DROP TABLE IF EXISTS employees CASCADE;

CREATE TABLE employees (
    id           INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name         TEXT NOT NULL,
    salary       NUMERIC(12,2) NOT NULL CHECK (salary >= 0),
    department   TEXT NOT NULL
);

INSERT INTO employees (name, salary, department) VALUES
('Анна',   50000, 'HR'),
('Иван',   80000, 'IT'),
('Мария',  60000, 'Finance'),
('Павел',  90000, 'IT'),
('Ольга',  45000, 'HR'),
('Дмитрий',75000, 'Finance');
```

**Контроль:**

```sql
SELECT department, COUNT(*) AS cnt, ROUND(AVG(salary)) AS avg_salary
FROM employees
GROUP BY department
ORDER BY department;
```

---

## 3. Таблица users (используется в Частях 2–5)

**Что демонстрируем:** триггеры `BEFORE/AFTER`, `OLD/NEW`, аудит, `updated_at`.

```sql
DROP TABLE IF EXISTS users CASCADE;

CREATE TABLE users (
    id          INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    username    TEXT UNIQUE NOT NULL,
    email       TEXT,
    updated_at  TIMESTAMP NOT NULL DEFAULT NOW()
);

-- Базовые данные
INSERT INTO users (username, email) VALUES
('alex',  'alex@mail.ru'),
('maria', 'maria@gmail.com'),
('ivan',  'ivan@yandex.ru'),
('kate',  'kate@corp.com');

-- Несколько изменений для историй и аудита
UPDATE users SET email = 'alex_new@mail.ru' WHERE username = 'alex';
UPDATE users SET email = LOWER(email) WHERE username IN ('maria', 'ivan');
```

**Контроль:**

```sql
SELECT * FROM users ORDER BY id;
```

---

## 4. История users_history (лог email; Часть 3)

**Что демонстрируем:** простой AFTER UPDATE-триггер, `OLD/NEW`, условие `WHEN`.

```sql
DROP TABLE IF EXISTS users_history;

CREATE TABLE users_history (
    id          INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    user_id     INT NOT NULL,
    old_email   TEXT,
    new_email   TEXT,
    changed_at  TIMESTAMP NOT NULL DEFAULT NOW()
);

-- Пример реального наполнения пойдёт от триггера;
-- но добавим 1 ручную строку для проверки формы данных:
INSERT INTO users_history (user_id, old_email, new_email, changed_at)
VALUES (1, 'alex@mail.ru', 'alex_new@mail.ru', NOW() - INTERVAL '1 day');
```

**Контроль:**

```sql
SELECT * FROM users_history ORDER BY id;
```

---

## 5. Универсальный JSONB-аудит audit_log (Части 3–5)

**Что демонстрируем:** `to_jsonb(OLD/NEW)`, `TG_OP`, `TG_TABLE_NAME`, `current_user`, `application_name`.

```sql
DROP TABLE IF EXISTS audit_log;

CREATE TABLE audit_log (
    id          INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    table_name  TEXT NOT NULL,
    operation   TEXT NOT NULL,                         -- INSERT / UPDATE / DELETE
    old_data    JSONB,
    new_data    JSONB,
    changed_at  TIMESTAMP NOT NULL DEFAULT NOW(),
    user_name   TEXT,
    app_name    TEXT
);

-- Небольшое тестовое наполнение (имитация INSERT/UPDATE/DELETE)
INSERT INTO audit_log (table_name, operation, old_data, new_data, user_name, app_name, changed_at)
VALUES
('users', 'INSERT', NULL, '{"id":10,"username":"test","email":"t@t.t"}', current_user, current_setting('application_name', true), NOW() - INTERVAL '2 days'),
('users', 'UPDATE', '{"id":10,"email":"t@t.t"}', '{"id":10,"email":"t2@t.t"}', current_user, current_setting('application_name', true), NOW() - INTERVAL '1 day'),
('users', 'DELETE', '{"id":10,"username":"test"}', NULL, current_user, current_setting('application_name', true), NOW() - INTERVAL '12 hours');
```

**Контроль:**

```sql
SELECT table_name, operation, user_name, app_name, changed_at
FROM audit_log
ORDER BY changed_at DESC;
```

---

## 6. Таблицы для бизнес-кейса заказов: products, orders (Часть 5)

**Что демонстрируем:** `BEFORE INSERT/UPDATE` рассчёт производного поля `total_price`, массовые апдейты, отчёты.

```sql
DROP TABLE IF EXISTS orders   CASCADE;
DROP TABLE IF EXISTS products CASCADE;

CREATE TABLE products (
    id     INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name   TEXT UNIQUE NOT NULL,
    price  NUMERIC(12,2) NOT NULL CHECK (price >= 0)
);

CREATE TABLE orders (
    id           INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    product      TEXT NOT NULL REFERENCES products(name) ON UPDATE CASCADE,
    quantity     INT NOT NULL CHECK (quantity > 0),
    total_price  NUMERIC(14,2),
    updated_at   TIMESTAMP NOT NULL DEFAULT NOW()
);

-- Наполним продукты
INSERT INTO products (name, price) VALUES
('Laptop',      120000.00),
('Keyboard',       4500.00),
('Mouse',          2500.00),
('Monitor',       30000.00),
('USB Hub',        1900.00);

-- Заказы (без триггера total_price пока будут NULL)
INSERT INTO orders (product, quantity) VALUES
('Laptop',  1),
('Mouse',   3),
('Keyboard',2),
('Monitor', 2),
('USB Hub', 4);

-- Для кейсов апдейтов
UPDATE products SET price = price * 1.05 WHERE name IN ('Laptop','Monitor');
```

**Контроль:**

```sql
SELECT o.id, o.product, o.quantity, o.total_price, o.updated_at
FROM orders o
ORDER BY o.id;
```

> Позже в части 5 на эту схему вешается триггер для расчёта `total_price`.

---

## 7. deleted_users (лог удаления; Часть 3 — самостоятелька)

**Что демонстрируем:** `AFTER DELETE` логирование в отдельную таблицу.

```sql
DROP TABLE IF EXISTS deleted_users;

CREATE TABLE deleted_users (
    id          INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    user_id     INT NOT NULL,
    email       TEXT,
    deleted_at  TIMESTAMP NOT NULL DEFAULT NOW()
);

-- Ручной пример записи
INSERT INTO deleted_users (user_id, email, deleted_at)
VALUES (3, 'ivan@yandex.ru', NOW() - INTERVAL '6 hours');
```

**Контроль:**

```sql
SELECT * FROM deleted_users ORDER BY id;
```

---

## 8. events (JSON/лог-событий — примеры JSONB/ClickHouse)

**Что демонстрируем:** хранение событий с гибкой схемой, JSON-агрегации/фильтры.

```sql
DROP TABLE IF EXISTS events;

CREATE TABLE events (
    id          INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    created_at  TIMESTAMP NOT NULL DEFAULT NOW(),
    data        JSONB NOT NULL
);

-- События логина/клика/покупки
INSERT INTO events (created_at, data) VALUES
(NOW() - INTERVAL '2 days',  '{"event":"login","user_id":101,"device":"ios"}'),
(NOW() - INTERVAL '1 day',   '{"event":"click","user_id":101,"button":"buy"}'),
(NOW() - INTERVAL '20 hours','{"event":"login","user_id":202,"device":"android"}'),
(NOW() - INTERVAL '12 hours','{"event":"purchase","user_id":101,"amount":1990.50,"currency":"RUB"}'),
(NOW() - INTERVAL '1 hour',  '{"event":"click","user_id":202,"button":"details"}');
```

**Контроль:**

```sql
SELECT
  data->>'event' AS event,
  COUNT(*) AS cnt
FROM events
GROUP BY data->>'event'
ORDER BY cnt DESC;
```

---

## 9. Вспомогательные таблицы для примеров архивации

**Что демонстрируем:** перенос старых данных в архив (используется в задачах со `*`).

```sql
DROP TABLE IF EXISTS orders_archive;

CREATE TABLE orders_archive (
    id           INT,
    product      TEXT,
    quantity     INT,
    total_price  NUMERIC(14,2),
    updated_at   TIMESTAMP,
    archived_at  TIMESTAMP NOT NULL DEFAULT NOW()
);

-- Немного старых данных для демонстрации
INSERT INTO orders_archive (id, product, quantity, total_price, updated_at, archived_at)
SELECT id, product, quantity, total_price, updated_at, NOW() - INTERVAL '1 day'
FROM orders
WHERE id IN (1,2);  -- допустим, архивировали пару самых ранних
```

**Контроль:**

```sql
SELECT COUNT(*) AS archived_cnt FROM orders_archive;
```

---

## 10. Быстрые проверки и полезные индексы

Ниже — необязательные, но полезные для практики индексы и запросы-проверки.

```sql
-- Индексы под частые фильтры/джоины
CREATE INDEX IF NOT EXISTS idx_users_username ON users (username);
CREATE INDEX IF NOT EXISTS idx_users_email    ON users (email);

CREATE INDEX IF NOT EXISTS idx_orders_product ON orders (product);
CREATE INDEX IF NOT EXISTS idx_products_name  ON products (name);

-- GIN-индекс для JSONB (если планируются частые фильтры по ключам)
-- CREATE INDEX IF NOT EXISTS idx_events_data_gin ON events USING gin (data);

-- Проверки

-- Средняя зарплата по отделам
SELECT department, ROUND(AVG(salary)) AS avg_salary
FROM employees
GROUP BY department
ORDER BY department;

-- Последние изменения в users (с учётом updated_at)
SELECT id, username, email, updated_at
FROM users
ORDER BY updated_at DESC
LIMIT 5;

-- Свежие записи аудита по таблице users
SELECT operation, user_name, app_name, changed_at
FROM audit_log
WHERE table_name = 'users'
ORDER BY changed_at DESC
LIMIT 5;

-- Сводка заказов и актуальная цена продукта
SELECT o.id, o.product, o.quantity, p.price AS current_price, o.total_price
FROM orders o
JOIN products p ON p.name = o.product
ORDER BY o.id;

-- Частота событий по типу
SELECT data->>'event' AS event, COUNT(*) AS cnt
FROM events
GROUP BY data->>'event'
ORDER BY cnt DESC;
```

---

### ✅ Что теперь есть “из коробки”

* `employees` — для функций с SELECT INTO / UPDATE / RETURN QUERY
* `users` — для триггеров, `updated_at`, аудита
* `users_history` — история изменений email
* `audit_log` — универсальный JSONB-аудит
* `products` + `orders` — бизнес-кейс с расчётом `total_price`
* `deleted_users` — лог удалений
* `events` — JSON/лог-событий для JSONB-практик
* `orders_archive` — для задач архивации

> На этот набор можно **накатывать все функции и триггеры** из файлов 01–07 без доработок структуры.


