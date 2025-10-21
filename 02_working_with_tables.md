## 📘 `02_working_with_tables.md`

# Работа с таблицами в PL/pgSQL

---

## 🧭 Оглавление

* [1. Введение: зачем работать с таблицами внутри функций](#1-введение-зачем-работать-с-таблицами-внутри-функций)
* [2. SELECT INTO — выборка данных в переменные](#2-select-into--выборка-данных-в-переменные)
* [3. INSERT, UPDATE, DELETE внутри функций](#3-insert-update-delete-внутри-функций)
* [4. PERFORM — выполнение без возврата данных](#4-perform--выполнение-без-возврата-данных)
* [5. RETURN QUERY — возврат набора строк](#5-return-query--возврат-набора-строк)
* [6. Контекстные переменные FOUND и ROW_COUNT](#6-контекстные-переменные-found-и-row_count)
* [7. Практическая задача: повышение зарплат по отделу](#7-практическая-задача-повышение-зарплат-по-отделу)
* [8. Задачи для самостоятельного выполнения](#8-задачи-для-самостоятельного-выполнения)

---

## 1. Введение: зачем работать с таблицами внутри функций

PL/pgSQL позволяет выполнять операции с таблицами напрямую —
добавлять, изменять, удалять и возвращать данные.

Это нужно, когда:

* нужно реализовать бизнес-логику прямо в БД;
* требуется аудит или обработка данных без клиента;
* нужно готовить отчёты и агрегаты на стороне сервера.

---

## 2. SELECT INTO — выборка данных в переменные

`SELECT INTO` сохраняет результат запроса в переменные.

```sql
CREATE OR REPLACE FUNCTION get_salary(emp_name TEXT)
RETURNS NUMERIC AS $$
DECLARE
    emp_salary NUMERIC;
BEGIN
    SELECT salary INTO emp_salary
    FROM employees
    WHERE name = emp_name;

    RETURN emp_salary;
END;
$$ LANGUAGE plpgsql;
```

🧩 **Пояснение:**

* `INTO emp_salary` записывает результат запроса в переменную.
* Если строка не найдена, переменная = `NULL`.
* После запроса можно проверить `IF FOUND THEN ...`.

**Проверка:**

```sql
SELECT get_salary('Анна');
SELECT get_salary('Неизвестный');
```

---

## 3. INSERT, UPDATE, DELETE внутри функций

Функции могут менять таблицы напрямую.

```sql
CREATE OR REPLACE FUNCTION update_email(u_id INT, new_email TEXT)
RETURNS TEXT AS $$
BEGIN
    UPDATE users
    SET email = new_email
    WHERE id = u_id;

    IF FOUND THEN
        RETURN 'Почта обновлена';
    ELSE
        RETURN 'Пользователь не найден';
    END IF;
END;
$$ LANGUAGE plpgsql;
```

🧩 **Пояснение:**

* `FOUND` — служебная переменная (TRUE, если обновлено хотя бы 1 строка).
* Для точного числа строк можно использовать `GET DIAGNOSTICS count = ROW_COUNT;`.

---

## 4. PERFORM — выполнение без возврата данных

Когда результат не нужен, применяют `PERFORM`.

```sql
CREATE OR REPLACE FUNCTION delete_empty_emails()
RETURNS VOID AS $$
BEGIN
    PERFORM 1 FROM users WHERE email IS NULL;
    DELETE FROM users WHERE email IS NULL;
END;
$$ LANGUAGE plpgsql;
```

🧩 **Пояснение:**
`PERFORM` работает как `SELECT`, но игнорирует результат — удобно при вызове функций без возврата.

---

## 5. RETURN QUERY — возврат набора строк

Если функция возвращает несколько строк — используется `RETURN QUERY`.

```sql
CREATE OR REPLACE FUNCTION get_department_employees(dep TEXT)
RETURNS TABLE(id INT, name TEXT, salary NUMERIC) AS $$
BEGIN
    RETURN QUERY
    SELECT id, name, salary
    FROM employees
    WHERE department = dep;
END;
$$ LANGUAGE plpgsql;
```

**Проверка:**

```sql
SELECT * FROM get_department_employees('IT');
```

Можно объединять несколько `RETURN QUERY` — все результаты объединятся.

---

## 6. Контекстные переменные FOUND и ROW_COUNT

После каждого SQL-оператора PostgreSQL сохраняет служебную информацию:

| Переменная        | Описание                                          |
| ----------------- | ------------------------------------------------- |
| `FOUND`           | TRUE, если последний запрос затронул ≥ 1 строку   |
| `ROW_COUNT`       | Количество строк, изменённых последним оператором |
| `GET DIAGNOSTICS` | Позволяет явно прочитать эти значения             |

Пример:

```sql
CREATE OR REPLACE FUNCTION mark_bonus(dep TEXT)
RETURNS TEXT AS $$
DECLARE
    affected INT;
BEGIN
    UPDATE employees
    SET salary = salary * 1.1
    WHERE department = dep;

    GET DIAGNOSTICS affected = ROW_COUNT;

    RETURN 'Изменено сотрудников: ' || affected;
END;
$$ LANGUAGE plpgsql;
```

---

## 7. Практическая задача: повышение зарплат по отделу

**Задача:**
Создай функцию `increase_salary(dep TEXT, percent NUMERIC)` —
увеличивает зарплату сотрудников отдела на `percent %` и возвращает количество обновлённых строк.

```sql
CREATE OR REPLACE FUNCTION increase_salary(dep TEXT, percent NUMERIC)
RETURNS INT AS $$
DECLARE
    updated_count INT;
BEGIN
    UPDATE employees
    SET salary = salary * (1 + percent / 100)
    WHERE department = dep;

    GET DIAGNOSTICS updated_count = ROW_COUNT;

    RETURN updated_count;
END;
$$ LANGUAGE plpgsql;
```

**Проверка:**

```sql
SELECT increase_salary('IT', 10);
SELECT * FROM employees;
```

🟢 Функция возвращает число изменённых строк, а зарплаты реально увеличиваются.

---

## 8. Задачи для самостоятельного выполнения

### 🧩 Уровень 1

Создай функцию `delete_low_salary(min NUMERIC)` —
удаляет сотрудников с зарплатой ниже `min` и возвращает количество удалённых строк.

---

### 🧩 Уровень 2

Создай функцию `copy_department(old_dep TEXT, new_dep TEXT)` —
копирует всех сотрудников из одного отдела в другой (изменяя поле `department`).

---

### 🧩 Уровень 3

Создай функцию `average_salary(dep TEXT)` —
возвращает среднюю зарплату по отделу.
Подсказка: `SELECT AVG(salary) INTO ...`.

---

### 🧩 Уровень 4

Создай функцию `employees_report()` —
возвращает таблицу `(department, count, avg_salary)`
через `RETURN QUERY` и `GROUP BY`.

---

✅ После этой части ты умеешь:

* работать с таблицами внутри функций;
* использовать `SELECT INTO`, `UPDATE`, `DELETE`, `RETURN QUERY`;
* применять служебные переменные `FOUND`, `ROW_COUNT`, `GET DIAGNOSTICS`;
* реализовывать бизнес-логику на стороне PostgreSQL.

---

📘 Следующий файл → [03_triggers_intro.md](03_triggers_intro.md)
**“Триггеры в PostgreSQL: OLD, NEW, TG_OP, TG_TABLE_NAME и аудит изменений”**
-----------------------------------------------------------------------------


