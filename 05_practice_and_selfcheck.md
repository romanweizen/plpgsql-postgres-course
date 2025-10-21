## 📘 `05_practice_and_selfcheck.md`

# Практика, проверка решений и самопроверка по курсу PL/pgSQL и триггерам

---

## 🧭 Оглавление

* [1. Введение и цели блока](#1-введение-и-цели-блока)
* [2. Итоговые практические задачи](#2-итоговые-практические-задачи)

  * [2.1 Аудит изменений в нескольких таблицах](#21-аудит-изменений-в-нескольких-таблицах)
  * [2.2 Автоматическое обновление поля updated_at](#22-автоматическое-обновление-поля-updated_at)
  * [2.3 Расчёт поля total_price при INSERT и UPDATE](#23-расчёт-поля-total_price-при-insert-и-update)
  * [2.4 Универсальный JSONB-аудит](#24-универсальный-jsonb-аудит)
  * [2.5 Вывод истории изменений для конкретной записи](#25-вывод-истории-изменений-для-конкретной-записи)
* [3. Разбор решений](#3-разбор-решений)
* [4. Вопросы для самопроверки](#4-вопросы-для-самопроверки)

---

## 1. Введение и цели блока

Этот блок объединяет всё, что ты изучил:

* создание функций на **PL/pgSQL**;
* работа с таблицами, условиями, циклами;
* использование контекстных переменных (`OLD`, `NEW`, `TG_OP`, `TG_TABLE_NAME`);
* создание триггеров и аудит через **JSONB**.

Здесь ты закрепишь навыки на полноценных задачах, которые встречаются в реальной практике и на собеседованиях на позиции **Data Engineer / DevOps / Backend Developer**.

---

## 2. Итоговые практические задачи

---

### 2.1 Аудит изменений в нескольких таблицах

**Задача:**
Создай единую систему аудита, которая отслеживает изменения в таблицах:

* `users`
* `employees`
* `products`

и записывает все операции (`INSERT`, `UPDATE`, `DELETE`) в одну таблицу `audit_log`.

**Подсказка:**
используй одну универсальную функцию и `TG_TABLE_NAME` для определения источника изменений.

---

### 2.2 Автоматическое обновление поля updated_at

**Задача:**
Создай триггер `BEFORE UPDATE`, который всегда обновляет поле `updated_at` в любой таблице, где это поле есть.
Пример — таблица `orders`:

```sql
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    product TEXT,
    quantity INT,
    total_price NUMERIC,
    updated_at TIMESTAMP DEFAULT NOW()
);
```

При каждом `UPDATE` поле `updated_at` должно автоматически обновляться на `NOW()`.

---

### 2.3 Расчёт поля total_price при INSERT и UPDATE

**Задача:**
Добавь триггер для таблицы `orders`, который:

* перед вставкой (`BEFORE INSERT`) или обновлением (`BEFORE UPDATE`)
* вычисляет `total_price = quantity * (SELECT price FROM products WHERE name = NEW.product)`

---

### 2.4 Универсальный JSONB-аудит

**Задача:**
Создай триггер `audit_all_tables` на основе функции `audit_changes_verbose()`,
который можно прикрепить к нескольким таблицам (`users`, `orders`, `products`).

> 💡 Подсказка: в функции используй `TG_TABLE_NAME`, `TG_OP`, `current_user`, `clock_timestamp()`
> и `to_jsonb(OLD)`, `to_jsonb(NEW)`.

---

### 2.5 Вывод истории изменений для конкретной записи

**Задача:**
Создай функцию `get_audit_history(table_name TEXT, record_id INT)`
которая возвращает JSON-массив всех изменений данной записи из таблицы `audit_log`.

Результат должен содержать список всех версий изменений (`old_data`, `new_data`, `changed_at`).

**Пример вызова:**

```sql
SELECT get_audit_history('users', 5);
```

🟢 Возврат:

```json
[
  {"changed_at": "2025-10-21T01:02:03", "old_email": "old@mail.ru", "new_email": "new@mail.ru"},
  {"changed_at": "2025-10-22T13:45:00", "old_email": "new@mail.ru", "new_email": "final@mail.ru"}
]
```

---

## 3. Разбор решений

---

### ✅ 2.1 Аудит изменений в нескольких таблицах

```sql
CREATE OR REPLACE FUNCTION audit_all()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO audit_log (table_name, operation, old_data, new_data, user_name)
    VALUES (
        TG_TABLE_NAME,
        TG_OP,
        to_jsonb(OLD),
        to_jsonb(NEW),
        current_user
    );
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

**Привязка:**

```sql
CREATE TRIGGER audit_users AFTER INSERT OR UPDATE OR DELETE ON users
FOR EACH ROW EXECUTE FUNCTION audit_all();

CREATE TRIGGER audit_employees AFTER INSERT OR UPDATE OR DELETE ON employees
FOR EACH ROW EXECUTE FUNCTION audit_all();

CREATE TRIGGER audit_products AFTER INSERT OR UPDATE OR DELETE ON products
FOR EACH ROW EXECUTE FUNCTION audit_all();
```

---

### ✅ 2.2 Автоматическое обновление updated_at

```sql
CREATE OR REPLACE FUNCTION refresh_updated_at()
RETURNS TRIGGER AS $$
BEGIN
    IF NEW.updated_at IS DISTINCT FROM OLD.updated_at THEN
        NEW.updated_at := NOW();
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER set_updated_at
BEFORE UPDATE ON orders
FOR EACH ROW EXECUTE FUNCTION refresh_updated_at();
```

---

### ✅ 2.3 Расчёт total_price

```sql
CREATE OR REPLACE FUNCTION calc_total_price()
RETURNS TRIGGER AS $$
DECLARE
    price NUMERIC;
BEGIN
    SELECT p.price INTO price FROM products p WHERE p.name = NEW.product;
    IF price IS NOT NULL THEN
        NEW.total_price := price * NEW.quantity;
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER set_total_price
BEFORE INSERT OR UPDATE ON orders
FOR EACH ROW EXECUTE FUNCTION calc_total_price();
```

---

### ✅ 2.4 Универсальный JSONB-аудит

```sql
CREATE OR REPLACE FUNCTION audit_verbose()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO audit_log (table_name, operation, old_data, new_data, user_name, app_name, changed_at)
    VALUES (
        TG_TABLE_NAME,
        TG_OP,
        to_jsonb(OLD),
        to_jsonb(NEW),
        current_user,
        current_setting('application_name', true),
        clock_timestamp()
    );
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

**Привязка к нескольким таблицам:**

```sql
CREATE TRIGGER audit_users AFTER INSERT OR UPDATE OR DELETE ON users
FOR EACH ROW EXECUTE FUNCTION audit_verbose();

CREATE TRIGGER audit_orders AFTER INSERT OR UPDATE OR DELETE ON orders
FOR EACH ROW EXECUTE FUNCTION audit_verbose();
```

---

### ✅ 2.5 История изменений

```sql
CREATE OR REPLACE FUNCTION get_audit_history(table_name TEXT, record_id INT)
RETURNS JSONB AS $$
DECLARE
    history JSONB;
BEGIN
    SELECT jsonb_agg(jsonb_build_object(
        'changed_at', changed_at,
        'old_data', old_data,
        'new_data', new_data
    ))
    INTO history
    FROM audit_log
    WHERE table_name = get_audit_history.table_name
    AND (old_data->>'id')::INT = record_id
       OR (new_data->>'id')::INT = record_id;

    RETURN history;
END;
$$ LANGUAGE plpgsql;
```

---

## 4. Вопросы для самопроверки

---

### 🔹 Теоретические

1. Чем отличаются `BEFORE` и `AFTER` триггеры?
2. Что такое `NEW` и `OLD` и где их нельзя использовать?
3. Что делает переменная `TG_OP`?
4. В каких случаях `RETURN NEW` обязателен?
5. Как работает `GET DIAGNOSTICS ROW_COUNT`?
6. Почему для аудита выгодно использовать JSONB?
7. Что делает выражение `to_jsonb(NEW) - to_jsonb(OLD)`?
8. Что делает оператор `IS DISTINCT FROM` и почему он лучше `!=`?

---

### 🔹 Практические

1. Как реализовать универсальный триггер для всех таблиц?
2. Как получить только изменённые поля между OLD и NEW?
3. Как в триггере определить, кто выполнил изменение?
4. Как добавить имя клиента (приложения) в лог?
5. Как безопасно логировать DELETE без NEW?

---

### 💡 Ответы для проверки

1. `BEFORE` — выполняется до изменения, можно модифицировать NEW.
   `AFTER` — выполняется после, подходит для логирования.
2. `NEW` — новая строка, есть при INSERT/UPDATE.
   `OLD` — старая строка, есть при UPDATE/DELETE.
3. `TG_OP` возвращает тип операции (`INSERT`, `UPDATE`, `DELETE`).
4. `RETURN NEW` обязателен для BEFORE-триггеров, чтобы сохранить строку.
5. `GET DIAGNOSTICS` извлекает служебные данные, включая `ROW_COUNT`.
6. JSONB гибок, поддерживает ключи произвольной структуры.
7. `to_jsonb(NEW) - to_jsonb(OLD)` — вычитает одинаковые поля, оставляя только изменённые.
8. `IS DISTINCT FROM` корректно работает с `NULL`, в отличие от `!=`.

---

✅ После этого блока ты готов:

* писать сложные функции и триггеры PL/pgSQL;
* проектировать систему аудита “из коробки”;
* уверенно отвечать на вопросы на собеседовании по PostgreSQL и ClickHouse.

---

📘 Следующий файл → [06_interview_questions.md](06_interview_questions.md)
**“Вопросы и ответы для подготовки к собеседованию по PostgreSQL и PL/pgSQL”**
