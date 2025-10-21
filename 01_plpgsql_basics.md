## 📘 `01_plpgsql_basics.md`

# Основы PL/pgSQL — функции, переменные, IF, циклы

---

## 🧭 Оглавление

* [1. Что такое PL/pgSQL](#1-что-такое-plpgsql)
* [2. Структура функции](#2-структура-функции)
* [3. Объявление переменных и констант](#3-объявление-переменных-и-констант)
* [4. Управляющие конструкции (IF, ELSIF, ELSE)](#4-управляющие-конструкции-if-elsif-else)
* [5. Циклы (LOOP, WHILE, FOR)](#5-циклы-loop-while-for)
* [6. Возврат данных (RETURN, RETURN QUERY)](#6-возврат-данных-return-return-query)
* [7. Работа с контекстными переменными](#7-работа-с-контекстными-переменными)
* [8. Практическая задача: оценка студента](#8-практическая-задача-оценка-студента)
* [9. Задачи для самостоятельного выполнения](#9-задачи-для-самостоятельного-выполнения)

---

## 1. Что такое PL/pgSQL

**PL/pgSQL (Procedural Language/PostgreSQL SQL)** — это встроенный процедурный язык в PostgreSQL.
Он позволяет создавать *функции, процедуры, триггеры* и *автоматизировать логику* внутри базы данных.

> Если SQL — это язык **что** нужно сделать,
> то PL/pgSQL — это язык **как именно это сделать**.

Основные применения:

* инкапсуляция бизнес-логики в БД;
* аудит и логирование изменений;
* автоматизация операций (триггеры, расчёты, проверки);
* валидация и обработка ошибок;
* работа с циклами и условиями внутри SQL.

---

## 2. Структура функции

Базовый шаблон функции PL/pgSQL:

```sql
CREATE OR REPLACE FUNCTION function_name(param_name param_type, ...)
RETURNS return_type AS $$
DECLARE
    -- объявление переменных
BEGIN
    -- тело функции
    RETURN значение;
END;
$$ LANGUAGE plpgsql;
```

Пример:

```sql
CREATE OR REPLACE FUNCTION say_hello(name TEXT)
RETURNS TEXT AS $$
BEGIN
    RETURN 'Привет, ' || name || '!';
END;
$$ LANGUAGE plpgsql;
```

Проверка:

```sql
SELECT say_hello('Алексей');
```

🟢 Результат:

```
Привет, Алексей!
```

---

## 3. Объявление переменных и констант

Переменные задаются в блоке `DECLARE`:

```sql
DECLARE
    total INT := 0;       -- с инициализацией
    name TEXT;            -- без значения
    pi CONSTANT NUMERIC := 3.14159;  -- константа
```

Использование:

```sql
CREATE OR REPLACE FUNCTION calc_square(radius NUMERIC)
RETURNS NUMERIC AS $$
DECLARE
    area NUMERIC;
    pi CONSTANT NUMERIC := 3.14159;
BEGIN
    area := pi * radius * radius;
    RETURN area;
END;
$$ LANGUAGE plpgsql;
```

---

## 4. Управляющие конструкции (IF, ELSIF, ELSE)

PL/pgSQL поддерживает условия как в языках программирования:

```sql
IF условие THEN
    ...
ELSIF другое_условие THEN
    ...
ELSE
    ...
END IF;
```

Пример:

```sql
CREATE OR REPLACE FUNCTION grade_score(score INT)
RETURNS TEXT AS $$
DECLARE
    grade TEXT;
BEGIN
    IF score >= 90 THEN
        grade := 'Отлично';
    ELSIF score >= 75 THEN
        grade := 'Хорошо';
    ELSIF score >= 60 THEN
        grade := 'Удовлетворительно';
    ELSE
        grade := 'Неудовлетворительно';
    END IF;
    RETURN grade;
END;
$$ LANGUAGE plpgsql;
```

Проверка:

```sql
SELECT grade_score(95);  -- Отлично
SELECT grade_score(78);  -- Хорошо
SELECT grade_score(40);  -- Неудовлетворительно
```

---

## 5. Циклы (LOOP, WHILE, FOR)

PL/pgSQL поддерживает три типа циклов:

### 🌀 5.1 LOOP

Бесконечный цикл, который прерывается через `EXIT`:

```sql
LOOP
    EXIT WHEN total > 100;
    total := total + 10;
END LOOP;
```

---

### 🌀 5.2 WHILE

```sql
WHILE условие LOOP
    ...
END LOOP;
```

---

### 🌀 5.3 FOR

```sql
FOR i IN 1..n LOOP
    ...
END LOOP;
```

Пример:

```sql
CREATE OR REPLACE FUNCTION sum_to_n(n INT)
RETURNS INT AS $$
DECLARE
    total INT := 0;
    i INT;
BEGIN
    FOR i IN 1..n LOOP
        total := total + i;
    END LOOP;
    RETURN total;
END;
$$ LANGUAGE plpgsql;
```

Проверка:

```sql
SELECT sum_to_n(10);
```

🟢 Результат: `55`

---

## 6. Возврат данных (RETURN, RETURN QUERY)

Функции могут возвращать одно значение (`RETURN`),
или таблицу (`RETURN QUERY`):

```sql
CREATE OR REPLACE FUNCTION get_it_department()
RETURNS TABLE(name TEXT, salary NUMERIC) AS $$
BEGIN
    RETURN QUERY
    SELECT name, salary FROM employees WHERE department = 'IT';
END;
$$ LANGUAGE plpgsql;
```

Проверка:

```sql
SELECT * FROM get_it_department();
```

---

## 7. Работа с контекстными переменными

PL/pgSQL предоставляет набор *встроенных переменных*, которые описывают текущее состояние выполнения функции или триггера.

| Переменная          | Контекст | Описание                                  |
| ------------------- | -------- | ----------------------------------------- |
| `current_user`      | любой    | имя текущего пользователя PostgreSQL      |
| `session_user`      | любой    | имя пользователя сессии                   |
| `NEW`               | триггер  | запись после изменения                    |
| `OLD`               | триггер  | запись до изменения                       |
| `TG_OP`             | триггер  | тип операции (INSERT / UPDATE / DELETE)   |
| `TG_TABLE_NAME`     | триггер  | имя таблицы, на которой сработал триггер  |
| `TG_WHEN`           | триггер  | момент срабатывания (BEFORE / AFTER)      |
| `TG_LEVEL`          | триггер  | уровень (ROW / STATEMENT)                 |
| `FOUND`             | функция  | true, если последний запрос вернул строки |
| `ROW_COUNT`         | функция  | количество изменённых строк               |
| `clock_timestamp()` | любой    | текущее время в момент вызова             |

> ⚡ Пример:
>
> * `NEW.email` — новое значение столбца после UPDATE.
> * `OLD.email` — старое значение столбца до UPDATE.
> * `TG_OP` = `'UPDATE'` позволяет в триггере понять, какое событие произошло.
> * `current_user` полезен для аудита: кто выполнял операцию.

---

## 8. Практическая задача: оценка студента

**Задача:**
Создай функцию `student_result(name TEXT, score INT)`,
которая возвращает текст:

> `<Имя>: <оценка>`
> где оценка рассчитывается как в `grade_score`.

**Решение:**

```sql
CREATE OR REPLACE FUNCTION student_result(name TEXT, score INT)
RETURNS TEXT AS $$
DECLARE
    grade TEXT;
BEGIN
    grade := grade_score(score);
    RETURN name || ': ' || grade;
END;
$$ LANGUAGE plpgsql;
```

**Проверка:**

```sql
SELECT student_result('Иван', 87);
```

🟢 → `Иван: Хорошо`

---

## 9. Задачи для самостоятельного выполнения

### 🧩 Уровень 1

Создай функцию `double_number(n INT)` — возвращает число, умноженное на 2.

### 🧩 Уровень 2

Создай функцию `even_or_odd(n INT)` — возвращает `"Чётное"` или `"Нечётное"`.

### 🧩 Уровень 3

Создай функцию `factorial(n INT)` — вычисляет факториал через цикл `FOR`.

---

## ✅ Следующая часть

👉 [02_working_with_tables.md](02_working_with_tables.md)
**"Работа с таблицами и возврат данных"**

---
