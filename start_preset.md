Отлично 💪
Вот готовый файл **`start_preset.md`** — оформлен в том же стиле, что и предыдущие части курса (`01_intro.md`, `02_working_with_tables.md` и т.д.).
Это единый установочный и стартовый пресет окружения: PostgreSQL + pg_cron + ClickHouse + Mongo + pgAdmin,
со всеми инструкциями, кодом и комментариями в удобном виде для копирования в проект.

---

```markdown
# 🧱 Start Preset: Docker-окружение для курса PL/pgSQL и ClickHouse

Этот файл содержит полностью готовую конфигурацию окружения для курса.  
Включены:
- PostgreSQL с расширением **pg_cron**
- ClickHouse
- MongoDB
- Опционально: pgAdmin
- Автоматическая инициализация схемы и заданий через `initdb`

---

## ⚙️ 1. Структура проекта

```

📦 plpgsql-postgres-course/
┣ 📂 postgres/
┃ ┗ 📜 Dockerfile
┣ 📂 initdb/
┃ ┣ 📜 00_create_extensions_and_schema.sql
┃ ┣ 📜 01_pgcron_jobs.sql
┃ ┗ 📜 02_seed_course.sql        ← (добавляется позже)
┣ 📜 docker-compose.yml
┣ 📜 DOCKER_README.md
┗ 📜 start_preset.md             ← этот файл

````

---

## 🐋 2. Docker Compose

```yaml
version: '3.8'

services:
  postgres:
    build:
      context: ./postgres
      dockerfile: Dockerfile
    container_name: plpgsql_postgres_course_db
    restart: always
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
      POSTGRES_DB: example_db
    ports:
      - "5433:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./initdb:/docker-entrypoint-initdb.d:ro
    command: >
      postgres -c shared_preload_libraries=pg_cron
               -c cron.database_name=example_db

  clickhouse:
    image: clickhouse/clickhouse-server:latest
    container_name: plpgsql_clickhouse_course_db
    restart: always
    ports:
      - "8124:8123"
      - "9001:9000"
    ulimits:
      nofile:
        soft: 262144
        hard: 262144
    environment:
      CLICKHOUSE_USER: user
      CLICKHOUSE_PASSWORD: strongpassword
      CLICKHOUSE_DEFAULT_ACCESS_MANAGEMENT: "1"
    volumes:
      - clickhouse_data:/var/lib/clickhouse

  mongo:
    image: mongo:6
    container_name: plpgsql_mongo_course_db
    restart: always
    ports:
      - "27018:27017"
    volumes:
      - mongodb_data:/data/db

  pgadmin:
    image: dpage/pgadmin4:latest
    container_name: plpgsql_pgadmin
    restart: unless-stopped
    environment:
      PGADMIN_DEFAULT_EMAIL: admin@local
      PGADMIN_DEFAULT_PASSWORD: admin
    ports:
      - "8081:80"
    volumes:
      - pgadmin_data:/var/lib/pgadmin

volumes:
  pgdata:
  clickhouse_data:
  mongodb_data:
  pgadmin_data:
````

> 💡 Порты изменены для совместимости:
>
> * PostgreSQL: **5433**
> * ClickHouse: **8124 / 9001**
> * MongoDB: **27018**
> * pgAdmin: **8081**

---

## 🧩 3. Dockerfile (PostgreSQL с pg_cron)

`postgres/Dockerfile`

```dockerfile
FROM postgres:15

USER root

RUN apt-get update \
 && apt-get install -y --no-install-recommends \
      ca-certificates \
      curl \
      gnupg \
 && rm -rf /var/lib/apt/lists/*

# Установка расширения pg_cron
RUN apt-get update \
 && apt-get install -y --no-install-recommends postgresql-15-cron \
 && rm -rf /var/lib/apt/lists/*

USER postgres
```

> ⚠️ Если пакет `postgresql-15-cron` не найден, подключи официальный репозиторий PostgreSQL apt:
> см. комментарии в Dockerfile.

---

## 🧠 4. Автоматическая инициализация (`initdb/`)

Создай папку `initdb/` и добавь следующие SQL-файлы.
Они выполняются **при первом запуске контейнера**.

### 🧾 `00_create_extensions_and_schema.sql`

```sql
CREATE SCHEMA IF NOT EXISTS course;
SET search_path TO course, public;

CREATE EXTENSION IF NOT EXISTS pg_cron;

-- Дополнительно:
-- CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

SET application_name = 'course_init';
```

---

### ⏰ `01_pgcron_jobs.sql`

```sql
-- Очистка старых записей аудита каждые сутки в 03:00
SELECT cron.unschedule(jobid)
FROM cron.job
WHERE jobid = 1;

SELECT cron.schedule('cleanup_audit_log', '0 3 * * *', $$
    DELETE FROM course.audit_log
    WHERE changed_at < now() - INTERVAL '90 days';
$$);
```

---

### 🌱 `02_seed_course.sql` *(добавим позже)*

Сюда добавляется полный код создания и наполнения таблиц (из `00_reset_all.md`).
Этот файл будет выполняться при старте контейнера и создаст все объекты базы.

---

## 🧭 5. Инструкции по запуску

### ▶️ 1. Собрать и запустить

```bash
docker compose build
docker compose up -d
```

### 🔗 2. Подключение через DBeaver

| Параметр | Значение   |
| -------- | ---------- |
| Host     | localhost  |
| Port     | 5433       |
| Database | example_db |
| User     | user       |
| Password | password   |

---

## 📋 6. Проверка pg_cron

Проверить, что расширение и задания созданы:

```sql
\dx
SELECT * FROM cron.job;
```

> Если таблицы `cron.job` нет, проверь параметры запуска:
> `shared_preload_libraries=pg_cron` и `cron.database_name=example_db`.

---

## 🧰 7. Полезные команды

```bash
# Логи контейнера Postgres
docker compose logs -f postgres

# Выполнить SQL внутри контейнера
docker exec -it plpgsql_postgres_course_db \
  psql -U user -d example_db -c "SELECT now();"

# Пересоздать контейнеры с чистыми данными
docker compose down -v
docker compose up -d --build
```

---

## 🗺️ 8. pgAdmin (опционально)

Открой в браузере:
👉 [http://localhost:8081](http://localhost:8081)
Введи логин и пароль:

```
Email: admin@local
Password: admin
```

Добавь новое подключение к серверу:

* Host: postgres
* Port: 5432 (внутренний порт контейнера)
* Database: example_db
* Username: user
* Password: password

---

## ⚡ 9. pg_cron: примеры расписаний

| Задача         | Cron-синтаксис           | Описание            |
| -------------- | ------------------------ | ------------------- |
| `0 0 * * *`    | Каждый день в полночь    | Ежедневная чистка   |
| `*/10 * * * *` | Каждые 10 минут          | Проверка активности |
| `30 2 * * 1`   | По понедельникам в 02:30 | Архивация данных    |

Пример задания:

```sql
SELECT cron.schedule(
  'rebuild_stats',
  '0 4 * * *',
  $$ VACUUM ANALYZE; $$
);
```

---

## 🧾 10. Сводка

| Компонент            | Назначение                | Порт        |
| -------------------- | ------------------------- | ----------- |
| PostgreSQL + pg_cron | основная БД курса         | 5433        |
| ClickHouse           | аналитика и сравнение     | 8124 / 9001 |
| MongoDB              | вспомогательные задачи    | 27018       |
| pgAdmin              | веб-управление PostgreSQL | 8081        |

---

## ✅ 11. Готовность к курсу

После выполнения команд:

```bash
docker compose build
docker compose up -d
```

ты получаешь полное окружение:

* PostgreSQL с активным pg_cron
* готовую схему `course`
* ClickHouse и Mongo для сравнения
* удобный доступ через DBeaver или pgAdmin

---

### 💬 Дальше

Следующий шаг — подключить `02_seed_course.sql`, чтобы автоматически создать все таблицы и наполнить их данными.

---

**Автор:**

> 🔹 Проект: `plpgsql-postgres-course`
> 🔹 Раздел: `start_preset.md`
> 🔹 Назначение: подготовка и запуск полного окружения для практики PL/pgSQL, pg_cron и ClickHouse.

```

---

Хочешь, я добавлю в этот же формат следующий файл — `02_seed_course.md`, который будет содержать весь код из `00_reset_all.md` в SQL-виде для автоматической инициализации (с `CREATE TABLE`, `INSERT` и т.д.)?  
Тогда при `docker compose up` всё создастся само.
```

