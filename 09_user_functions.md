# 09_user_functions.md

## Пользовательские функции, триггеры и pg_cron в PostgreSQL

---

## 📁 Оглавление

1. [🧬 Введение](#-введение)
2. [🗺️ Схема потока данных (ASCII)](#%EF%B8%8F-%D1%81%D1%85%D0%B5%D0%BC%D0%B0-%D0%BF%D0%BE%D1%82%D0%BE%D0%BA%D0%B0-%D0%B4%D0%B0%D0%BD%D0%BD%D1%8B%D1%85-ascii)
3. [1. Функции и процедуры: базовые понятия](#1-%D1%84%D1%83%D0%BD%D0%BA%D1%86%D0%B8%D0%B8-%D0%B8-%D0%BF%D1%80%D0%BE%D1%86%D0%B5%D0%B4%D1%83%D1%80%D1%8B-%D0%B1%D0%B0%D0%B7%D0%BE%D0%B2%D1%8B%D0%B5-%D0%BF%D0%BE%D0%BD%D1%8F%D1%82%D0%B8%D1%8F)
4. [2. Виды функций в PostgreSQL](#2-%D0%B2%D0%B8%D0%B4%D1%8B-%D1%84%D1%83%D0%BD%D0%BA%D1%86%D0%B8%D0%B9-%D0%B2-postgresql)
5. [3. Простая SQL-функция](#3-%D0%BF%D1%80%D0%BE%D1%81%D1%82%D0%B0%D1%8F-sql-%D1%84%D1%83%D0%BD%D0%BA%D1%86%D0%B8%D1%8F)
6. [4. Функции на PLpgSQL](#4-%D1%84%D1%83%D0%BD%D0%BA%D1%86%D0%B8%D0%B8-%D0%BD%D0%B0-plpgsql)
7. [5. Условные операторы и циклы](#5-%D1%83%D1%81%D0%BB%D0%BE%D0%B2%D0%BD%D1%8B%D0%B5-%D0%BE%D0%BF%D0%B5%D1%80%D0%B0%D1%82%D0%BE%D1%80%D1%8B-%D0%B8-%D1%86%D0%B8%D0%BA%D0%BB%D1%8B)
8. [6. Триггеры и триггерные функции](#6-%D1%82%D1%80%D0%B8%D0%B3%D0%B3%D0%B5%D1%80%D1%8B-%D0%B8-%D1%82%D1%80%D0%B8%D0%B3%D0%B3%D0%B5%D1%80%D0%BD%D1%8B%D0%B5-%D1%84%D1%83%D0%BD%D0%BA%D1%86%D0%B8%D0%B8)
9. [7. Планировщик задач pg_cron](#7-%D0%BF%D0%BB%D0%B0%D0%BD%D0%B8%D1%80%D0%BE%D0%B2%D1%89%D0%B8%D0%BA-%D0%B7%D0%B0%D0%B4%D0%B0%D1%87-pg_cron)
10. [8. Практика](#8-%D0%BF%D1%80%D0%B0%D0%BA%D1%82%D0%B8%D0%BA%D0%B0)
11. [9. Контрольные и интервью-вопросы](#9-%D0%BA%D0%BE%D0%BD%D1%82%D1%80%D0%BE%D0%BB%D1%8C%D0%BD%D1%8B%D0%B5-%D0%B8-%D0%B8%D0%BD%D1%82%D0%B5%D1%80%D0%B2%D1%8C%D1%8E-%D0%B2%D0%BE%D0%BF%D1%80%D0%BE%D1%81%D1%8B)
12. [🧩 Итог](#-itog)

---

### 🧬 Введение

**Функции** — подпрограммы, выполняющие набор операций и возвращающие результат.

🔹 Они позволяют повторно использовать код, сократить дублирование запросов, инкапсулировать бизнес-логику и автоматизировать задачи.

---

### 🗺️ Схема потока данных (ASCII)

```
[Client/App] → users → Trigger → users_audit → pg_cron export CSV
```

---

### 1. Функции и процедуры

| Тип       | Возврат                | Вызов    | Транзакции     |
| --------- | ---------------------- | -------- | -------------- |
| FUNCTION  | Обязателен (`RETURNS`) | `SELECT` | Не коммитит    |
| PROCEDURE | Необязателен (`OUT`)   | `CALL`   | Может `COMMIT` |

---

### 2. Виды функций

* Встроенные (AVG, SUM, NOW)
* Пользовательские (user-defined)
* SQL, PL/pgSQL, C, Python, Perl и др.

---

### 3. Простая SQL-функция

```sql
CREATE OR REPLACE FUNCTION get_user_count()
RETURNS INTEGER AS $$ SELECT COUNT(*) FROM users; $$ LANGUAGE SQL;
SELECT get_user_count();
```

Функция с параметром:

```sql
CREATE OR REPLACE FUNCTION get_user_by_id(p_id INT)
RETURNS TEXT AS $$ SELECT name FROM users WHERE id=p_id; $$ LANGUAGE SQL;
SELECT get_user_by_id(5);
```

---

### 4. Функции на PLpgSQL

```sql
CREATE OR REPLACE FUNCTION get_user_role(p_id INT)
RETURNS TEXT AS $$
DECLARE v_role TEXT;
BEGIN
 SELECT role INTO v_role FROM users WHERE id=p_id;
 IF v_role IS NULL THEN RETURN 'Не найден'; ELSE RETURN v_role; END IF;
END; $$ LANGUAGE plpgsql;
```

Возврат таблицы:

```sql
CREATE OR REPLACE FUNCTION get_users_by_role(p_role TEXT)
RETURNS TABLE(id INT,name TEXT,email TEXT) AS $$
BEGIN RETURN QUERY SELECT id,name,email FROM users WHERE role=p_role; END;
$$ LANGUAGE plpgsql;
```

---

### 5. Условные операторы и циклы

```sql
IF ... THEN ... ELSIF ... ELSE ... END IF;
CASE WHEN ... THEN ... ELSE ... END CASE;
FOR i IN 1..10 LOOP RAISE NOTICE 'Итерация %',i; END LOOP;
```

---

### 6. Триггеры и триггерные функции

```sql
CREATE OR REPLACE FUNCTION log_user_update()
RETURNS TRIGGER AS $$
BEGIN
 IF (OLD.name IS DISTINCT FROM NEW.name)
  OR (OLD.email IS DISTINCT FROM NEW.email)
  OR (OLD.role IS DISTINCT FROM NEW.role) THEN
    INSERT INTO users_audit(user_id,field_changed,old_value,new_value,changed_at)
    VALUES (OLD.id,'user_data',OLD.name||','||OLD.email||','||OLD.role,
            NEW.name||','||NEW.email||','||NEW.role,CURRENT_TIMESTAMP);
 END IF;
 RETURN NEW;
END; $$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_log_user_changes
BEFORE UPDATE ON users
FOR EACH ROW EXECUTE FUNCTION log_user_update();
```

---

### 7. Планировщик задач pg_cron

```sql
CREATE EXTENSION IF NOT EXISTS pg_cron;
SELECT cron.schedule('0 2 * * *', $$CALL export_new_users_to_csv();$$);

CREATE OR REPLACE PROCEDURE export_new_users_to_csv()
LANGUAGE plpgsql AS $$
BEGIN
 COPY (SELECT * FROM users_audit WHERE changed_at>NOW()-INTERVAL '1 day')
 TO '/exports/users_audit.csv' WITH CSV HEADER;
END; $$;
```

---

### 8. Практика

* Создай функцию `get_user_count_by_role(role)`
* Создай триггер на удаления → `users_deleted`
* Cron-задача очистки аудита >7 дней

Пример:

```sql
CREATE OR REPLACE FUNCTION get_user_count_by_role(p_role TEXT)
RETURNS INTEGER AS $$ SELECT COUNT(*) FROM users WHERE role=p_role; $$ LANGUAGE SQL;
```

---

### 9. Контрольные и интервью-вопросы

1. FUNCTION vs PROCEDURE?
2. Зачем ограничители `$$`?
3. RETURN QUERY — что делает?
4. Разница RETURN NEW / RETURN OLD?
5. Для чего pg_cron?
6. Почему нельзя COMMIT в функции?
7. Где посмотреть расширения?

---

### 🧩 Итог

* Функции — инкапсулированная логика SQL.
* Процедуры — действия без прямого возврата.
* Триггеры — автоматическая реакция на изменения.
* `pg_cron` — автоматизация внутри PostgreSQL.

---

**Далее:** → [10_scheduling_and_automation.md](10_scheduling_and_automation.md)
