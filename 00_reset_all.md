
## 📘 `00_reset_all.md`

# Полный ресет окружения курса: пересоздание схемы, таблиц и тестовых данных

---

## 🧭 Оглавление

* [1. Что делает этот скрипт](#1-что-делает-этот-скрипт)
* [2. Быстрый старт](#2-быстрый-старт)
* [3. Полный скрипт ресета (скопируй и выполни целиком)](#3-полный-скрипт-ресета-скопируй-и-выполни-целиком)
* [4. Что создано после ресета](#4-что-создано-после-ресета)
* [5. Санити-чеки (быстрые проверки)](#5-санити-чеки-быстрые-проверки)
* [6. Примечания и тонкости](#6-примечания-и-тонкости)

---

## 1. Что делает этот скрипт

* создаёт (или пересоздаёт) **схему `course`** и ставит `search_path` → `course, public`;
* **чистит** следы прошлых запусков: таблицы, триггеры, функции, чтобы не мешали;
* создаёт **все таблицы**, используемые во всём курсе:

  * `employees`, `users`, `users_history`, `audit_log`,
    `products`, `orders`, `deleted_users`, `events`, `orders_archive`;
* **заполняет** таблицы тестовыми данными, согласованными со всеми примерами;
* добавляет **полезные индексы** (по e-mail, username, product и т.д.);
* готовит тебя к немедленному запуску кодов из файлов `01–08`.

---

## 2. Быстрый старт

1. Открой DBeaver/psql → выбери нужную БД.
2. Скопируй **весь** блок из раздела №3.
3. Выполни его целиком.
4. Запусти проверки из раздела №5 — увидишь ожидаемые результаты.

---

## 3. Полный скрипт ресета (скопируй и выполни целиком)

```sql
-- ===========================================================
-- PL/pgSQL Course: Full Reset Script
-- Пересоздание схемы, таблиц и тестовых данных
-- Совместим с PostgreSQL 13+
-- ===========================================================

-- 0) Общие настройки
SET client_min_messages = WARNING;

-- 1) Создаём чистую схему и ставим search_path
DO $$
BEGIN
  IF NOT EXISTS (
    SELECT 1 FROM pg_namespace WHERE nspname = 'course'
  ) THEN
    EXECUTE 'CREATE SCHEMA course';
  END IF;
END $$;

SET search_path TO course, public;

-- 2) Удаляем функции из прошлых запусков (если были)
--    (чтобы сигнатуры не мешали, перечисляем наиболее частые из курса)
DO $cleanup$
BEGIN
  -- Базовые PL/pgSQL (могли быть созданы в учебных шагах)
  PERFORM 1 FROM pg_proc WHERE proname = 'say_hello';                 IF FOUND THEN EXECUTE 'DROP FUNCTION IF EXISTS say_hello(text) CASCADE'; END IF;
  PERFORM 1 FROM pg_proc WHERE proname = 'grade_score';               IF FOUND THEN EXECUTE 'DROP FUNCTION IF EXISTS grade_score(integer) CASCADE'; END IF;
  PERFORM 1 FROM pg_proc WHERE proname = 'sum_to_n';                  IF FOUND THEN EXECUTE 'DROP FUNCTION IF EXISTS sum_to_n(integer) CASCADE'; END IF;
  PERFORM 1 FROM pg_proc WHERE proname = 'get_salary';                IF FOUND THEN EXECUTE 'DROP FUNCTION IF EXISTS get_salary(text) CASCADE'; END IF;
  PERFORM 1 FROM pg_proc WHERE proname = 'update_email';              IF FOUND THEN EXECUTE 'DROP FUNCTION IF EXISTS update_email(integer,text) CASCADE'; END IF;
  PERFORM 1 FROM pg_proc WHERE proname = 'delete_empty_emails';       IF FOUND THEN EXECUTE 'DROP FUNCTION IF EXISTS delete_empty_emails() CASCADE'; END IF;
  PERFORM 1 FROM pg_proc WHERE proname = 'get_department_employees';  IF FOUND THEN EXECUTE 'DROP FUNCTION IF EXISTS get_department_employees(text) CASCADE'; END IF;
  PERFORM 1 FROM pg_proc WHERE proname = 'mark_bonus';                IF FOUND THEN EXECUTE 'DROP FUNCTION IF EXISTS mark_bonus(text) CASCADE'; END IF;
  PERFORM 1 FROM pg_proc WHERE proname = 'increase_salary';           IF FOUND THEN EXECUTE 'DROP FUNCTION IF EXISTS increase_salary(text,numeric) CASCADE'; END IF;

  -- Триггерные функции и аудит
  PERFORM 1 FROM pg_proc WHERE proname = 'update_timestamp';          IF FOUND THEN EXECUTE 'DROP FUNCTION IF EXISTS update_timestamp() CASCADE'; END IF;
  PERFORM 1 FROM pg_proc WHERE proname = 'log_user_update';           IF FOUND THEN EXECUTE 'DROP FUNCTION IF EXISTS log_user_update() CASCADE'; END IF;
  PERFORM 1 FROM pg_proc WHERE proname = 'audit_changes';             IF FOUND THEN EXECUTE 'DROP FUNCTION IF EXISTS audit_changes() CASCADE'; END IF;
  PERFORM 1 FROM pg_proc WHERE proname = 'audit_changes_verbose';     IF FOUND THEN EXECUTE 'DROP FUNCTION IF EXISTS audit_changes_verbose() CASCADE'; END IF;
  PERFORM 1 FROM pg_proc WHERE proname = 'audit_differences';         IF FOUND THEN EXECUTE 'DROP FUNCTION IF EXISTS audit_differences() CASCADE'; END IF;
  PERFORM 1 FROM pg_proc WHERE proname = 'audit_update_delete';       IF FOUND THEN EXECUTE 'DROP FUNCTION IF EXISTS audit_update_delete() CASCADE'; END IF;
  PERFORM 1 FROM pg_proc WHERE proname = 'refresh_updated_at';        IF FOUND THEN EXECUTE 'DROP FUNCTION IF EXISTS refresh_updated_at() CASCADE'; END IF;
  PERFORM 1 FROM pg_proc WHERE proname = 'calc_total_price';          IF FOUND THEN EXECUTE 'DROP FUNCTION IF EXISTS calc_total_price() CASCADE'; END IF;
  PERFORM 1 FROM pg_proc WHERE proname = 'audit_all';                 IF FOUND THEN EXECUTE 'DROP FUNCTION IF EXISTS audit_all() CASCADE'; END IF;
  PERFORM 1 FROM pg_proc WHERE proname = 'audit_verbose';             IF FOUND THEN EXECUTE 'DROP FUNCTION IF EXISTS audit_verbose() CASCADE'; END IF;
  PERFORM 1 FROM pg_proc WHERE proname = 'get_audit_history';         IF FOUND THEN EXECUTE 'DROP FUNCTION IF EXISTS get_audit_history(text,integer) CASCADE'; END IF;
  PERFORM 1 FROM pg_proc WHERE proname = 'soft_delete';               IF FOUND THEN EXECUTE 'DROP FUNCTION IF EXISTS soft_delete() CASCADE'; END IF;
  PERFORM 1 FROM pg_proc WHERE proname = 'audit_changed_columns';     IF FOUND THEN EXECUTE 'DROP FUNCTION IF EXISTS audit_changed_columns() CASCADE'; END IF;
  PERFORM 1 FROM pg_proc WHERE proname = 'archive_old_records';       IF FOUND THEN EXECUTE 'DROP FUNCTION IF EXISTS archive_old_records() CASCADE'; END IF;

  -- Динамика и пр.
  PERFORM 1 FROM pg_proc WHERE proname = 'dynamic_select';            IF FOUND THEN EXECUTE 'DROP FUNCTION IF EXISTS dynamic_select(text) CASCADE'; END IF;
END
$cleanup$ LANGUAGE plpgsql;

-- 3) Сносим таблицы (если остались) и пересоздаём
DROP TABLE IF EXISTS orders_archive CASCADE;
DROP TABLE IF EXISTS events         CASCADE;
DROP TABLE IF EXISTS deleted_users  CASCADE;
DROP TABLE IF EXISTS orders         CASCADE;
DROP TABLE IF EXISTS products       CASCADE;
DROP TABLE IF EXISTS audit_log      CASCADE;
DROP TABLE IF EXISTS users_history  CASCADE;
DROP TABLE IF EXISTS users          CASCADE;
DROP TABLE IF EXISTS employees      CASCADE;

-- 3.1) employees
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

-- 3.2) users
CREATE TABLE users (
    id          INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    username    TEXT UNIQUE NOT NULL,
    email       TEXT,
    updated_at  TIMESTAMP NOT NULL DEFAULT NOW()
);

INSERT INTO users (username, email) VALUES
('alex',  'alex@mail.ru'),
('maria', 'maria@gmail.com'),
('ivan',  'ivan@yandex.ru'),
('kate',  'kate@corp.com');

UPDATE users SET email = 'alex_new@mail.ru' WHERE username = 'alex';
UPDATE users SET email = LOWER(email) WHERE username IN ('maria', 'ivan');

-- 3.3) users_history
CREATE TABLE users_history (
    id          INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    user_id     INT NOT NULL,
    old_email   TEXT,
    new_email   TEXT,
    changed_at  TIMESTAMP NOT NULL DEFAULT NOW()
);

INSERT INTO users_history (user_id, old_email, new_email, changed_at)
VALUES (1, 'alex@mail.ru', 'alex_new@mail.ru', NOW() - INTERVAL '1 day');

-- 3.4) audit_log
CREATE TABLE audit_log (
    id          INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    table_name  TEXT NOT NULL,
    operation   TEXT NOT NULL,
    old_data    JSONB,
    new_data    JSONB,
    changed_at  TIMESTAMP NOT NULL DEFAULT NOW(),
    user_name   TEXT,
    app_name    TEXT
);

INSERT INTO audit_log (table_name, operation, old_data, new_data, user_name, app_name, changed_at)
VALUES
('users', 'INSERT', NULL, '{"id":10,"username":"test","email":"t@t.t"}', current_user, current_setting('application_name', true), NOW() - INTERVAL '2 days'),
('users', 'UPDATE', '{"id":10,"email":"t@t.t"}', '{"id":10,"email":"t2@t.t"}', current_user, current_setting('application_name', true), NOW() - INTERVAL '1 day'),
('users', 'DELETE', '{"id":10,"username":"test"}', NULL, current_user, current_setting('application_name', true), NOW() - INTERVAL '12 hours');

-- 3.5) products / orders
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

INSERT INTO products (name, price) VALUES
('Laptop',      120000.00),
('Keyboard',       4500.00),
('Mouse',          2500.00),
('Monitor',       30000.00),
('USB Hub',        1900.00);

INSERT INTO orders (product, quantity) VALUES
('Laptop',  1),
('Mouse',   3),
('Keyboard',2),
('Monitor', 2),
('USB Hub', 4);

UPDATE products SET price = price * 1.05 WHERE name IN ('Laptop','Monitor');

-- 3.6) deleted_users
CREATE TABLE deleted_users (
    id          INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    user_id     INT NOT NULL,
    email       TEXT,
    deleted_at  TIMESTAMP NOT NULL DEFAULT NOW()
);

INSERT INTO deleted_users (user_id, email, deleted_at)
VALUES (3, 'ivan@yandex.ru', NOW() - INTERVAL '6 hours');

-- 3.7) events
CREATE TABLE events (
    id          INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    created_at  TIMESTAMP NOT NULL DEFAULT NOW(),
    data        JSONB NOT NULL
);

INSERT INTO events (created_at, data) VALUES
(NOW() - INTERVAL '2 days',  '{"event":"login","user_id":101,"device":"ios"}'),
(NOW() - INTERVAL '1 day',   '{"event":"click","user_id":101,"button":"buy"}'),
(NOW() - INTERVAL '20 hours','{"event":"login","user_id":202,"device":"android"}'),
(NOW() - INTERVAL '12 hours','{"event":"purchase","user_id":101,"amount":1990.50,"currency":"RUB"}'),
(NOW() - INTERVAL '1 hour',  '{"event":"click","user_id":202,"button":"details"}');

-- 3.8) orders_archive
CREATE TABLE orders_archive (
    id           INT,
    product      TEXT,
    quantity     INT,
    total_price  NUMERIC(14,2),
    updated_at   TIMESTAMP,
    archived_at  TIMESTAMP NOT NULL DEFAULT NOW()
);

INSERT INTO orders_archive (id, product, quantity, total_price, updated_at, archived_at)
SELECT id, product, quantity, total_price, updated_at, NOW() - INTERVAL '1 day'
FROM orders
WHERE id IN (1,2);

-- 4) Полезные индексы
CREATE INDEX IF NOT EXISTS idx_users_username ON users (username);
CREATE INDEX IF NOT EXISTS idx_users_email    ON users (email);
CREATE INDEX IF NOT EXISTS idx_orders_product ON orders (product);
CREATE INDEX IF NOT EXISTS idx_products_name  ON products (name);
-- JSONB GIN индекс (включи при интенсивных JSON-запросах)
-- CREATE INDEX IF NOT EXISTS idx_events_data_gin ON events USING gin (data);

-- 5) Финальные установки
SET application_name = 'course_ready';
RESET client_min_messages;
-- Готово.
```

---

## 4. Что создано после ресета

* **Схема:** `course` (+ `search_path` выставлен)
* **Таблицы:**
  `employees`, `users`, `users_history`, `audit_log`,
  `products`, `orders`, `deleted_users`, `events`, `orders_archive`
* **Данные:** референсные наборы, согласующиеся с примерами из `01–08`
* **Индексы:** на частые фильтры; опциональный GIN для JSONB

---

## 5. Санити-чеки (быстрые проверки)

Сразу после выполнения ресета — прогоняй:

```sql
-- Свод по сотрудникам
SELECT department, COUNT(*) AS cnt, ROUND(AVG(salary)) AS avg_salary
FROM employees
GROUP BY department
ORDER BY department;

-- Актуальные пользователи (после апдейтов)
SELECT * FROM users ORDER BY id;

-- История e-mail
SELECT * FROM users_history ORDER BY changed_at DESC;

-- Тестовые записи аудита
SELECT table_name, operation, user_name, app_name, changed_at
FROM audit_log
ORDER BY changed_at DESC;

-- Заказы + актуальная цена продукта
SELECT o.id, o.product, o.quantity, p.price AS current_price, o.total_price
FROM orders o
JOIN products p ON p.name = o.product
ORDER BY o.id;

-- Частота JSON-событий
SELECT data->>'event' AS event, COUNT(*) AS cnt
FROM events
GROUP BY data->>'event'
ORDER BY cnt DESC;

-- Архив проверка
SELECT COUNT(*) AS archived_cnt FROM orders_archive;
```

---

## 6. Примечания и тонкости

* Скрипт **не** включает сами триггеры/функции из глав — они создаются в соответствующих файлах (например, `03_triggers_intro.md`, `04_jsonb_audit.md`, `05_practice_and_selfcheck.md`).
* Если ты уже навешивал триггеры в прошлых сессиях — удаление таблиц с `CASCADE` снимает и триггеры, а блок «очистки функций» сверху удаляет сами функции (чтобы не конфликтовали сигнатуры).
* Для работы с JSONB на больших массивах данных включай GIN-индекс на `events.data`.
* `SET search_path TO course, public` позволяет не префиксовать каждую таблицу `course.` во всех примерах.

---

Готово! Этот файл — твой «кнопка ресета».
Если хочешь, добавлю мини-скрипт `Makefile` или `psql -f` команду в README, чтобы ресет запускался одной командой.
