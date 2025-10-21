## 📘 `04_jsonb_audit.md`

# JSONB-аудит и продвинутые механизмы логирования в PostgreSQL

---

## 🧭 Оглавление

* [1. Зачем использовать JSONB для аудита](#1-зачем-использовать-jsonb-для-аудита)
* [2. Преимущества JSONB по сравнению с обычными таблицами истории](#2-преимущества-jsonb-по-сравнению-с-обычными-таблицами-истории)
* [3. Создание универсальной таблицы аудита](#3-создание-универсальной-таблицы-аудита)
* [4. Функция-триггер audit_changes() — базовая версия](#4-функция-триггер-audit_changes--базовая-версия)
* [5. Добавление контекста: пользователь, приложение, время](#5-добавление-контекста-пользователь-приложение-время)
* [6. Продвинутая версия — логирование только изменённых полей](#6-продвинутая-версия--логирование-только-изменённых-полей)
* [7. Практическая задача: гибридный аудит UPDATE и DELETE](#7-практическая-задача-гибридный-аудит-update-и-delete)
* [8. Задачи для самостоятельного выполнения](#8-задачи-для-самостоятельного-выполнения)

---

## 1. Зачем использовать JSONB для аудита

При классическом аудите создаются отдельные таблицы вроде `users_history`, `orders_history` и т.д.
Но когда таблиц становится десятки, это неудобно.

JSONB позволяет хранить **старые и новые версии строк в одной универсальной таблице**:

* ✅ Универсальность — одна структура для всех таблиц.
* ✅ Гибкость — можно хранить любую структуру без ALTER TABLE.
* ✅ Аналитика — можно выполнять запросы по ключам JSON (`old_data->>'email'`).
* ✅ Простота интеграции — JSON легко читать и экспортировать во внешние системы.

---

## 2. Преимущества JSONB по сравнению с обычными таблицами истории

| Критерий         | Таблица истории          | JSONB-аудит                              |
| ---------------- | ------------------------ | ---------------------------------------- |
| Гибкость схемы   | Жёсткая структура        | Динамическая                             |
| Расширяемость    | Нужно ALTER TABLE        | Можно добавлять ключи в JSON             |
| Скорость вставки | Высокая                  | Чуть ниже, но стабильная                 |
| Читаемость       | Хорошая, но разрозненная | Унифицированный формат                   |
| Интеграция       | Локальная                | Легко отправить в Kafka, ELK, ClickHouse |

---

## 3. Создание универсальной таблицы аудита

Создадим таблицу `audit_log`, в которую будем записывать все изменения в JSONB-формате.

```sql
CREATE TABLE audit_log (
    id SERIAL PRIMARY KEY,
    table_name TEXT,
    operation TEXT,
    old_data JSONB,
    new_data JSONB,
    changed_at TIMESTAMP DEFAULT NOW(),
    user_name TEXT,
    app_name TEXT
);
```

🧩 **Пояснение:**

* `old_data` — данные до изменения;
* `new_data` — данные после;
* `operation` — `INSERT`, `UPDATE` или `DELETE`;
* `app_name` и `user_name` сохраняют контекст.

---

## 4. Функция-триггер audit_changes() — базовая версия

```sql
CREATE OR REPLACE FUNCTION audit_changes()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO audit_log (table_name, operation, old_data, new_data, user_name, app_name)
    VALUES (
        TG_TABLE_NAME,
        TG_OP,
        to_jsonb(OLD),
        to_jsonb(NEW),
        current_user,
        current_setting('application_name', true)
    );
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

**Привязка триггера:**

```sql
CREATE TRIGGER audit_users
AFTER INSERT OR UPDATE OR DELETE ON users
FOR EACH ROW
EXECUTE FUNCTION audit_changes();
```

**Проверка:**

```sql
INSERT INTO users (username, email) VALUES ('ivan', 'ivan@mail.ru');
UPDATE users SET email = 'ivan_new@mail.ru' WHERE username = 'ivan';
DELETE FROM users WHERE username = 'ivan';

SELECT table_name, operation, old_data, new_data, user_name, app_name FROM audit_log;
```

🟢 В таблице `audit_log` появятся три строки — для каждой операции.

---

## 5. Добавление контекста: пользователь, приложение, время

PostgreSQL предоставляет встроенные функции, которые делают лог точнее:

| Функция                                     | Описание                                            |
| ------------------------------------------- | --------------------------------------------------- |
| `current_user`                              | имя пользователя, выполняющего операцию             |
| `session_user`                              | имя пользователя текущей сессии                     |
| `current_setting('application_name', true)` | имя приложения (например, DBeaver, Python, Airflow) |
| `clock_timestamp()`                         | точное время выполнения                             |

Пример с контекстом:

```sql
CREATE OR REPLACE FUNCTION audit_changes_verbose()
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

Теперь каждая запись содержит имя пользователя, клиента и точное время операции.

---

## 6. Продвинутая версия — логирование только изменённых полей

Иногда полный JSON избыточен.
Можно логировать **только те поля, которые действительно изменились**.

```sql
CREATE OR REPLACE FUNCTION audit_differences()
RETURNS TRIGGER AS $$
DECLARE
    diffs JSONB;
BEGIN
    -- вычисляем разницу между NEW и OLD
    diffs := (
        SELECT jsonb_object_agg(key, value)
        FROM jsonb_each(to_jsonb(NEW))
        WHERE (to_jsonb(OLD)->key) IS DISTINCT FROM value
    );

    IF diffs IS NOT NULL AND diffs <> '{}'::jsonb THEN
        INSERT INTO audit_log (table_name, operation, new_data, user_name)
        VALUES (TG_TABLE_NAME, TG_OP, diffs, current_user);
    END IF;

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

**Привязка:**

```sql
CREATE TRIGGER audit_users_diffs
AFTER UPDATE ON users
FOR EACH ROW
EXECUTE FUNCTION audit_differences();
```

**Проверка:**

```sql
UPDATE users SET email = 'new_email@mail.ru', username = 'alex' WHERE id = 1;
SELECT new_data FROM audit_log ORDER BY id DESC LIMIT 1;
```

🟢 Результат:

```json
{"email": "new_email@mail.ru", "username": "alex"}
```

---

## 7. Практическая задача: гибридный аудит UPDATE и DELETE

**Задача:**
Создать триггер, который логирует:

* для `UPDATE` — только изменённые поля (`diffs`);
* для `DELETE` — полные старые данные (`OLD`).

**Решение:**

```sql
CREATE OR REPLACE FUNCTION audit_update_delete()
RETURNS TRIGGER AS $$
DECLARE
    diffs JSONB;
BEGIN
    IF TG_OP = 'UPDATE' THEN
        diffs := (
            SELECT jsonb_object_agg(key, value)
            FROM jsonb_each(to_jsonb(NEW))
            WHERE (to_jsonb(OLD)->key) IS DISTINCT FROM value
        );

        IF diffs IS NOT NULL AND diffs <> '{}'::jsonb THEN
            INSERT INTO audit_log (table_name, operation, new_data, user_name)
            VALUES (TG_TABLE_NAME, 'UPDATE', diffs, current_user);
        END IF;

    ELSIF TG_OP = 'DELETE' THEN
        INSERT INTO audit_log (table_name, operation, old_data, user_name)
        VALUES (TG_TABLE_NAME, 'DELETE', to_jsonb(OLD), current_user);
    END IF;

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

**Привязка:**

```sql
CREATE TRIGGER audit_users_adv
AFTER UPDATE OR DELETE ON users
FOR EACH ROW
EXECUTE FUNCTION audit_update_delete();
```

**Проверка:**

```sql
UPDATE users SET email = 'delta@mail.ru' WHERE id = 1;
DELETE FROM users WHERE id = 1;

SELECT operation, old_data, new_data FROM audit_log ORDER BY id DESC LIMIT 2;
```

---

## 8. Задачи для самостоятельного выполнения

### 🧩 Уровень 1

Создай таблицу `products` и добавь универсальный триггер `audit_changes()`.
Проверь, что операции `INSERT`, `UPDATE`, `DELETE` логируются в `audit_log`.

---

### 🧩 Уровень 2

Измени функцию `audit_differences()` так, чтобы она записывала также `changed_at` и `app_name`.

---

### 🧩 Уровень 3

Создай функцию `audit_user_actions()`, которая записывает только те операции, где изменился email, и добавляет в JSON поле `"changed_field": "email"`.

---

### 🧩 Уровень 4

Реализуй универсальный триггер `audit_all_tables`, который можно повесить на несколько таблиц (users, employees, orders).
Подсказка: используй `TG_TABLE_NAME` и `TG_OP` — они автоматически передаются.

---

✅ После этой главы ты умеешь:

* строить универсальные системы аудита через JSONB;
* вычислять разницу между OLD и NEW;
* логировать только изменённые поля;
* сохранять контекст пользователя и приложения;
* комбинировать несколько типов операций в одном триггере.

---

📘 Следующий файл → [05_practice_and_selfcheck.md](05_practice_and_selfcheck.md)
**“Итоговая практика, разбор решений и самопроверка”**

