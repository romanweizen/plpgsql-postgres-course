# 🧠 PL/pgSQL & PostgreSQL Course

### Полный практический курс: SQL • PL/pgSQL • JSONB • Триггеры • Аудит • ClickHouse (в дальнейшем)

---

## 🧭 Оглавление

* [1. О курсе 🇷🇺](#1-о-курсе-🇷🇺)
* [2. Структура репозитория](#2-структура-репозитория)
* [3. Инструменты](#3-инструменты)
* [4. Как использовать](#4-как-использовать)
* [5. Навыки после прохождения](#5-навыки-после-прохождения)
* [6. PostgreSQL vs ClickHouse — кратко](#6-postgresql-vs-clickhouse--кратко)
* [7. Планы развития](#7-планы-развития)
* [8. Как внести вклад](#8-как-внести-вклад)
* [9. Лицензия](#9-лицензия)
* [10. English version 🇬🇧](#10-english-version-🇬🇧)

---

## 1. О курсе 🇷🇺

**`plpgsql-postgres-course`** — это учебный репозиторий для глубокого изучения **PostgreSQL** и **языка PL/pgSQL**,
через реальные инженерные задачи: функции, триггеры, JSONB-аудит, оптимизацию и переход к ClickHouse.

Формат — инженерная лаборатория:

* Каждый `.md` — отдельный модуль
* Примеры SQL и задачи с разбором
* Полная совместимость с VS Code Markdown Preview

---

## 2. Структура репозитория

| №   | Файл                           | Раздел                                                |
| --- | ------------------------------ | ----------------------------------------------------- |
| 1️⃣ | `01_plpgsql_basics.md`         | Основы PL/pgSQL, переменные, IF, LOOP, RETURN         |
| 2️⃣ | `02_working_with_tables.md`    | Работа с таблицами, SELECT INTO, RETURN QUERY         |
| 3️⃣ | `03_triggers_intro.md`         | Триггеры, OLD/NEW, TG_OP, TG_TABLE_NAME               |
| 4️⃣ | `04_jsonb_audit.md`            | JSONB-аудит, сравнение OLD и NEW                      |
| 5️⃣ | `05_practice_and_selfcheck.md` | Практика и разбор решений                             |
| 6️⃣ | `06_interview_questions.md`    | Вопросы и ответы для собеседований                    |
| 7️⃣ | `07_bonus_modern_features.md`  | Новые возможности PostgreSQL 15–17, pg_cron, pgvector |
| 8️⃣ | `08_clickhouse_comparison.md`  | PostgreSQL vs ClickHouse — архитектура и SQL-примеры  |

---

## 3. Инструменты

* PostgreSQL 15+
* DBeaver / psql
* VS Code
* Docker (для локальных тестов)
* Расширения: `pg_cron`, `pg_stat_statements`, `pgvector`, `pgaudit`

---

## 4. Как использовать

### 🔹 Клонировать репозиторий

```bash
git clone https://github.com/<yourname>/plpgsql-postgres-course.git
cd plpgsql-postgres-course
```

### 🔹 Открыть в VS Code

* Нажми **Ctrl + Shift + V** для предпросмотра Markdown
* Изучи файлы по порядку: `01_` → `08_`

### 🔹 (опционально) Создай ветку под ClickHouse

```bash
git checkout -b clickhouse_course
```

---

## 5. Навыки после прохождения

✅ Написание функций на PL/pgSQL
✅ Работа с контекстными переменными (`OLD`, `NEW`, `TG_OP`)
✅ Проектирование триггеров и JSONB-аудита
✅ Понимание архитектуры PostgreSQL vs ClickHouse
✅ Использование современных фич PostgreSQL 17+ (`MERGE`, `pg_cron`, `pgvector`)

---

## 6. PostgreSQL vs ClickHouse — кратко

| Категория           | PostgreSQL           | ClickHouse                |
| ------------------- | -------------------- | ------------------------- |
| Тип системы         | OLTP                 | OLAP                      |
| Хранение            | Строчное (row-store) | Колоночное (column-store) |
| Транзакции          | ACID                 | Нет                       |
| Аудит               | Триггеры             | Materialized Views        |
| JSON                | JSONB                | JSON / Nested / Map       |
| Основное применение | Web, CRM, финансы    | BI, аналитика, логи       |

---

## 7. Планы развития

* [ ] Ветка `clickhouse_course/` с практикой по ClickHouse
* [ ] Docker Compose для PostgreSQL + ClickHouse + DBeaver
* [ ] Финальный проект: “универсальный аудит + аналитика”
* [ ] Модули по расширениям (`pg_cron`, `pgvector`, `timescaledb`)

---

## 8. Как внести вклад

1. Форкни репозиторий
2. Создай ветку `feature/<тема>`
3. Добавь улучшения, новые задачи или комментарии
4. Сделай Pull Request 💬

---

## 9. Лицензия

MIT License — свободное использование и модификация
при указании авторства и ссылки на репозиторий.

---

## 10. English version 🇬🇧

---

### 📘 About

**`plpgsql-postgres-course`** is an open-source learning repository
for mastering **PostgreSQL and PL/pgSQL** through real-world engineering tasks:
functions, triggers, JSONB audit, optimization, and ClickHouse transition.

---

### 🧭 Repository Structure

| #   | File                           | Topic                                                  |
| --- | ------------------------------ | ------------------------------------------------------ |
| 1️⃣ | `01_plpgsql_basics.md`         | PL/pgSQL basics, variables, IF, LOOP, RETURN           |
| 2️⃣ | `02_working_with_tables.md`    | Working with tables, SELECT INTO, RETURN QUERY         |
| 3️⃣ | `03_triggers_intro.md`         | Triggers, OLD/NEW, TG_OP, TG_TABLE_NAME                |
| 4️⃣ | `04_jsonb_audit.md`            | JSONB audit and comparison of OLD vs NEW               |
| 5️⃣ | `05_practice_and_selfcheck.md` | Practice tasks and solutions                           |
| 6️⃣ | `06_interview_questions.md`    | Interview questions and concise answers                |
| 7️⃣ | `07_bonus_modern_features.md`  | PostgreSQL 15–17+ features, pg_cron, pgvector          |
| 8️⃣ | `08_clickhouse_comparison.md`  | PostgreSQL vs ClickHouse — architecture & SQL examples |

---

### ⚙️ Tools

* PostgreSQL 15+
* DBeaver / psql
* VS Code
* Docker (optional)
* Extensions: `pg_cron`, `pgvector`, `pg_stat_statements`

---

### 🚀 How to Use

```bash
git clone https://github.com/<yourname>/plpgsql-postgres-course.git
cd plpgsql-postgres-course
```

Open any `.md` file in **VS Code** and press `Ctrl + Shift + V` to preview the formatted material.

---

### 🎯 Learning Outcomes

* Write functions in PL/pgSQL
* Work with triggers and context variables
* Design JSONB-based audit systems
* Understand PostgreSQL vs ClickHouse architecture
* Apply modern PostgreSQL 17+ features

---

### 🧱 Future Plans

* [ ] ClickHouse practical course (new branch)
* [ ] Docker Compose setup (PostgreSQL + ClickHouse + DBeaver)
* [ ] Final project: universal audit + analytics
* [ ] Modules on extensions (`pg_cron`, `pgvector`, `timescaledb`)

---

### 💬 Contributing

Fork → Branch → Commit → Pull Request
⭐ Advanced SQL examples are welcome!

---

### 📎 License

MIT License — free to use, modify, and distribute with attribution.

---

> 🧠 *“A good data engineer doesn’t just write SQL — they think in data.”*

---

