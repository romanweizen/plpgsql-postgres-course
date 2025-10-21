Отлично 🚀
Вот готовая, полностью отформатированная третья часть курса —
**`03_triggers_intro.md`**
в том же стиле, что и предыдущие главы.
Открой новый файл в VS Code → вставь текст → сохрани как `03_triggers_intro.md`
и включи предпросмотр (**Ctrl + Shift + V**) — всё будет красиво и кликабельно.

---

## 📘 `03_triggers_intro.md`

# Триггеры в PostgreSQL — OLD, NEW, TG_OP, TG_TABLE_NAME и аудит изменений

---

## 🧭 Оглавление

* [1. Что такое триггер и зачем он нужен](#1-что-такое-триггер-и-зачем-он-нужен)
* [2. Типы триггеров и их назначение](#2-типы-триггеров-и-их-назначение)
* [3. Контекстные переменные триггеров (OLD, NEW, TG_OP и другие)](#3-контекстные-переменные-триггеров-old-new-tg_op-и-другие)
* [4. Пример: автоматическое обновление поля updated_at](#4-пример-автоматическое-обновление-поля-updated_at)
* [5. Пример: логирование изменений email в таблице users](#5-пример-логирование-изменений-email-в-таблице-users)
* [6. Разбор: разница между BEFORE и AFTER](#6-разбор-разница-между-before-и-after)
* [7. Практическая задача: аудит INSERT/UPDATE/DELETE](#7-практическая-задача-аудит-insertupdatedelete)
* [8. Задачи для самостоятельного выполнения](#8-задачи-для-самостоятельного-выполнения)

---

## 1. Что такое триггер и зачем он нужен

**Триггер (trigger)** — это механизм PostgreSQL, который автоматически выполняет заданную функцию при наступлении определённого события:
`INSERT`, `UPDATE`, `DELETE` или `TRUNCATE`.

> 💡 Триггеры позволяют “реагировать” на изменения в таблице без участия приложения.
> Это фундамент для аудита, версионирования, логирования и валидации.

Основные применения:

* аудит и логирование изменений;
* автоматическая валидация данных перед записью;
* обновление связанных таблиц;
* каскадные операции (например, пересчёт агрегатов).

---

## 2. Типы триггеров и их назначение

| Тип             | Когда срабатывает | Пример использования            |
| --------------- | ----------------- | ------------------------------- |
| `BEFORE INSERT` | до вставки        | изменить значения перед записью |
| `AFTER INSERT`  | после вставки     | логирование действий            |
| `BEFORE UPDATE` | до обновления     | обновить поля (`updated_at`)    |
| `AFTER UPDATE`  | после обновления  | аудит изменений                 |
| `BEFORE DELETE` | до удаления       | контроль удалений               |
| `AFTER DELETE`  | после удаления    | логирование удалённых строк     |

Дополнительно:

* `FOR EACH ROW` — триггер для каждой строки (наиболее часто).
* `FOR EACH STATEMENT` — триггер для всего оператора (`UPDATE employees SET ...`).

---

## 3. Контекстные переменные триггеров (OLD, NEW, TG_OP и другие)

| Переменная      | Тип    | Описание                                                   |
| --------------- | ------ | ---------------------------------------------------------- |
| `NEW`           | RECORD | новое состояние строки после операции (`INSERT`, `UPDATE`) |
| `OLD`           | RECORD | старое состояние строки до операции (`UPDATE`, `DELETE`)   |
| `TG_OP`         | TEXT   | тип операции (`INSERT`, `UPDATE`, `DELETE`, `TRUNCATE`)    |
| `TG_TABLE_NAME` | TEXT   | имя таблицы, где сработал триггер                          |
| `TG_WHEN`       | TEXT   | момент вызова (`BEFORE` или `AFTER`)                       |
| `TG_LEVEL`      | TEXT   | уровень (`ROW` или `STATEMENT`)                            |
| `current_user`  | TEXT   | имя пользователя, выполнившего действие                    |
| `session_user`  | TEXT   | имя пользователя сессии (часто совпадает с `current_user`) |

🔹 Пример внутри триггерной функции:

```sql
RAISE NOTICE 'Операция: %, Таблица: %, Пользователь: %', TG_OP, TG_TABLE_NAME, current_user;
```

🟢 Это выведет в консоль:
`Операция: UPDATE, Таблица: users, Пользователь: postgres`

---

## 4. Пример: автоматическое обновление поля updated_at

**Задача:** при любом изменении строки обновлять поле `updated_at`.

```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    username TEXT,
    email TEXT,
    updated_at TIMESTAMP DEFAULT NOW()
);
```

**Функция-триггер:**

```sql
CREATE OR REPLACE FUNCTION update_timestamp()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at := NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

**Привязка триггера:**

```sql
CREATE TRIGGER set_updated_at
BEFORE UPDATE ON users
FOR EACH ROW
EXECUTE FUNCTION update_timestamp();
```

🟢 Теперь `updated_at` всегда актуален без участия приложения.

---

## 5. Пример: логирование изменений email в таблице users

**Создадим таблицу истории:**

```sql
CREATE TABLE users_history (
    id SERIAL PRIMARY KEY,
    user_id INT,
    old_email TEXT,
    new_email TEXT,
    changed_at TIMESTAMP DEFAULT NOW()
);
```

**Функция-триггер:**

```sql
CREATE OR REPLACE FUNCTION log_user_update()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO users_history (user_id, old_email, new_email)
    VALUES (OLD.id, OLD.email, NEW.email);
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

**Привязка:**

```sql
CREATE TRIGGER user_update_trigger
AFTER UPDATE ON users
FOR EACH ROW
WHEN (OLD.email IS DISTINCT FROM NEW.email)
EXECUTE FUNCTION log_user_update();
```

**Проверка:**

```sql
INSERT INTO users (username, email) VALUES ('alex', 'a@mail.ru');
UPDATE users SET email = 'alex_new@mail.ru' WHERE username = 'alex';
SELECT * FROM users_history;
```

🟢 В `users_history` появится запись с `old_email` и `new_email`.

---

## 6. Разбор: разница между BEFORE и AFTER

| Параметр         | BEFORE                        | AFTER                  |
| ---------------- | ----------------------------- | ---------------------- |
| Время выполнения | до изменения данных           | после изменения данных |
| Можно менять NEW | ✅ да                          | ❌ нет                  |
| Доступ к OLD     | при UPDATE, DELETE            | при UPDATE, DELETE     |
| Типичные задачи  | модификация данных, валидация | аудит, логирование     |

💡 **Вывод:**

* Если нужно изменить данные → используй `BEFORE`.
* Если нужно зафиксировать факт изменения → используй `AFTER`.

---

## 7. Практическая задача: аудит INSERT/UPDATE/DELETE

Создадим универсальную таблицу аудита и триггер, который фиксирует все изменения в `users`.

```sql
CREATE TABLE audit_log (
    id SERIAL PRIMARY KEY,
    table_name TEXT,
    operation TEXT,
    old_data JSONB,
    new_data JSONB,
    changed_at TIMESTAMP DEFAULT NOW(),
    user_name TEXT
);
```

**Функция:**

```sql
CREATE OR REPLACE FUNCTION audit_changes()
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
CREATE TRIGGER audit_users
AFTER INSERT OR UPDATE OR DELETE ON users
FOR EACH ROW
EXECUTE FUNCTION audit_changes();
```

**Проверка:**

```sql
INSERT INTO users (username, email) VALUES ('maria', 'maria@mail.ru');
UPDATE users SET email = 'maria_new@mail.ru' WHERE username = 'maria';
DELETE FROM users WHERE username = 'maria';

SELECT table_name, operation, old_data, new_data, user_name FROM audit_log;
```

🟢 Результат: три записи — `INSERT`, `UPDATE`, `DELETE` с JSONB-полями и пользователем.

---

## 8. Задачи для самостоятельного выполнения

### 🧩 Уровень 1

Создай триггер, который при удалении строки из `users` записывает в таблицу `deleted_users` только `id` и `email`.

---

### 🧩 Уровень 2

Создай триггер `BEFORE INSERT` для таблицы `users`, который автоматически приводит email к нижнему регистру (`LOWER(email)`).

---

### 🧩 Уровень 3

Создай триггер, который при каждом `UPDATE` сохраняет изменения в универсальную таблицу `audit_log`,
но только если изменились конкретные поля — например, `email` или `username`.

---

### 🧩 Уровень 4

Создай триггер `AFTER INSERT` для таблицы `employees`,
который добавляет запись в `log_table` с временем и пользователем, создавшим сотрудника.

---

✅ После этой части ты умеешь:

* различать типы триггеров (`BEFORE`, `AFTER`);
* использовать `OLD`, `NEW`, `TG_OP`, `TG_TABLE_NAME`;
* реализовывать аудит и логирование;
* работать с JSONB внутри триггеров.

---

📘 Следующий файл → [04_jsonb_audit.md](04_jsonb_audit.md)
**“JSONB-аудит и продвинутые механизмы логирования в PostgreSQL”**

