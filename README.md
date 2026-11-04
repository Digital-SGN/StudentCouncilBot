<p align="center">
  <img src="https://github.com/Digital-SGN/Handbook/blob/main/assets/logo_%20council_bot.png" width="160" alt="Логотип" />
</p>

<h1 align="center"> Бот студенческого совета СГН </h1>
<p align="center"> 
  Telegram-бот для студенческого совета с функционалом регистрации на мероприятия.
<p/>

<p align="center">
  <img src="https://img.shields.io/badge/Python-3.12-3776AB?style=flat-square&logo=python&logoColor=white" />
  <img src="https://img.shields.io/badge/aiogram-3.x-2CABEE?style=flat-square&logo=telegram&logoColor=white" />
  <img src="https://img.shields.io/badge/PostgreSQL-16-4169E1?style=flat-square&logo=postgresql&logoColor=white" />
  <img src="https://img.shields.io/badge/Docker-ready-2496ED?style=flat-square&logo=docker&logoColor=white" />
</p>

## О проекте

Проводится много мероприятий в год, и на каждое — новая форма и ручная работа с таблицами. Бот нужен, чтобы автоматизировать регистрацию, напоминания и учёт явки.

**Основные возможности:**
- <img src="https://cdn.simpleicons.org/telegram/0CBFA1" width="16" /> **Регистрация:** один раз вводит данные (ФИО, группа, согласие на ПД).
- <img src="https://cdn.simpleicons.org/googlecalendar/0CBFA1" width="16" /> **Запись:** видит список активных мероприятий и записывается в один клик.
- <img src="https://cdn.simpleicons.org/clockify/0CBFA1" width="16" /> **Напоминания:** автоматические уведомления за 24 и за 2 часа до события.
- <img src="https://cdn.simpleicons.org/openbadges/0CBFA1" width="16" /> **Админ-панель:** команды для просмотра списков и отметки пришедших.


## Стек технологий

**Backend:**
- <img src="https://cdn.simpleicons.org/python/0CBFA1" width="16" /> Python 3.12 + aiogram 3.x
- <img src="https://cdn.simpleicons.org/postgresql/0CBFA1" width="16" /> PostgreSQL 16 (общая с платформой)
- <img src="https://cdn.simpleicons.org/sqlite/0CBFA1" width="16" /> SQLite + APScheduler (для напоминаний)

**Инфраструктура:**
- <img src="https://cdn.simpleicons.org/docker/0CBFA1" width="16" /> Docker + Docker Compose
- <img src="https://cdn.simpleicons.org/wireguard/0CBFA1" width="16" /> SSH-туннель (для связи с БД)


## Архитектура

```mermaid
flowchart LR
    User([Гость в Telegram])
    Admin([Админ])
    
    subgraph VPS ["VPS (Ubuntu 22.04)"]
        subgraph Docker ["Docker Network"]
            Bot["aiogram Bot"]
            SQLite[("SQLite")]
        end
        DB[("PostgreSQL")]
    end

    User <-->|Сообщения| Bot
    Admin <-->|Команды| Bot
    Bot <-->|APScheduler| SQLite
    Bot <-->|asyncpg / SQLAlchemy| DB
    
    classDef user fill:#fef3c7,stroke:#d97706,stroke-width:2px,color:#78350f;
    classDef bot fill:#ccfbf1,stroke:#0d9488,stroke-width:2px,color:#115e59;
    classDef db fill:#e0e7ff,stroke:#6366f1,stroke-width:2px,color:#312e81;
    classDef sqlite fill:#f1f5f9,stroke:#64748b,stroke-width:2px,color:#334155;

    class User,Admin user;
    class Bot bot;
    class DB db;
    class SQLite sqlite;
```


## Схема данных (PostgreSQL)

**guests:**
```sql
id — SERIAL PRIMARY KEY
telegram_id — BIGINT NOT NULL UNIQUE
username — TEXT
first_name — TEXT NOT NULL
last_name — TEXT NOT NULL
group_name — TEXT
consent_given_at — TIMESTAMPTZ NOT NULL
is_blocked — BOOLEAN DEFAULT FALSE
created_at — TIMESTAMPTZ DEFAULT NOW()
```

**registrations:**
```sql
id — SERIAL PRIMARY KEY
guest_id — INTEGER NOT NULL REFERENCES guests(id) ON DELETE CASCADE
event_id — INTEGER NOT NULL REFERENCES events(id) ON DELETE RESTRICT
status — TEXT NOT NULL DEFAULT 'registered' (значения: registered, cancelled, attended)
created_at — TIMESTAMPTZ DEFAULT NOW()
attended_at — TIMESTAMPTZ
UNIQUE (guest_id, event_id)
```


## Подключение к БД

Пользователь БД: `bot_user`.

Доступные таблицы:
- `SELECT, INSERT, UPDATE, DELETE` — `"Guests"`, `"Registrations"`
- `SELECT` — `"Events"`

Подключение к серверу выполняется через SSH-туннель, пользователь `dev_bot`.

В отдельном терминале запустить:
```bash
ssh -L 5433:localhost:5432 dev_bot@153.80.246.36
```

### Настройки DBeaver
- **Host:** `localhost`
- **Port:** `5433`
- **Database:** `studsovet`
- **Username:** `bot_user`
- **Password:** пароль от `bot_user`

### Переменные окружения (`.env`)

**Локально:**
```env
# Токен бота
BOT_TOKEN=123456789:AAHdqTcvCH1vGWJxfSeofSAs0K5PALDsaw

# База данных 
# Локально (через SSH-туннель):
DATABASE_URL=postgresql+asyncpg://bot_user:ваш_пароль@localhost:5433/studsovet
# В Docker (внутри сети):
# DATABASE_URL=postgresql+asyncpg://bot_user:ваш_пароль@studentcouncil_db:5432/studsovet

# Интеграция с Платформой (API)
API_BASE_URL=http://localhost:8080/api

# 4. Telegram ID админов через запятую (без пробелов), кому доступны команды /list и /attended
ADMINS=123456789,987654321

# 5. Настройки планировщика (Команда Scheduler)
# Часовой пояс для корректной отправки напоминаний (за 24ч и 2ч)
TIMEZONE=Europe/Moscow

# Версия согласия на обработку ПД, которую сейчас принимает бот
CONSENT_VERSION=1.0
```

**Docker:**
```env
DATABASE_URL=postgresql+asyncpg://bot_user:ПАРОЛЬ@studentcouncil_db:5432/studsovet
```

---

## Структура проекта

```text
bot/
├── handlers/
│   ├── __init__.py
│   ├── start.py          — /start, регистрация (FSM)
│   ├── events.py         — список меро, запись/отмена
│   ├── my.py             — /my, свои регистрации
│   └── admin.py          — /list, /attended
├── keyboards/
│   ├── __init__.py
│   ├── main.py           — главное меню
│   ├── events.py         — список меро, кнопки записи
│   └── admin.py          — админ-кнопки
├── middlewares/
│   ├── __init__.py
│   └── logging.py        — логирование апдейтов
├── db/
│   ├── __init__.py
│   ├── models.py         — SQLAlchemy: Guest, GuestRegistration
│   ├── session.py        — движок, sessionmaker
│   └── queries.py        — функции работы с БД
├── scheduler/
│   ├── __init__.py
│   ├── setup.py          — инициализация APScheduler + SQLite
│   └── reminders.py      — планирование и отправка напоминаний
├── data/
│   └── scheduler.db      — SQLite для APScheduler (volume)
├── config.py             — pydantic Settings, чтение .env
├── main.py               — точка входа: Bot, Dispatcher, polling
├── requirements.txt
├── Dockerfile
├── docker-compose.yml
└── .env                  — токен, ADMINS, DATABASE_URL
```


## Функционал MVP

**Гость:**
- `/start`  - FSM: имя → фамилия → группа → согласие
- Список активных меро (IsDeleted = false AND Status = 'Upcoming')
- «Записаться» - INSERT в registrations
- «Отменить» - registrations.status = "cancelled"
- `/my` — свои регистрации

**Админ** (список в .env):
- `/list <event_id>` — кто записался
- `/attended <guest_id> <event_id>` — отметить пришедшего


## Напоминания

- При регистрации — 2 задачи в APScheduler: за 24ч и за 2ч.
- Задачи хранятся в SQLite на VPS бота (volume).
- При отмене записи — обе задачи удаляются.
- При изменении даты меро — задачи пересоздаются.
- `misfire_grace_time=3600` — если бот лежал, догонит в течение часа.
- Перед отправкой — проверка, что регистрация активна.


## Что критично с самого начала

1. Отдельный пользователь БД с ограниченными правами (`bot_user`).
2. `UNIQUE (guest_id, event_id)` — защита от двойного клика.
3. `ON DELETE CASCADE` на guests, `RESTRICT` на events.
4. Согласие на ПД фиксируется при регистрации.
5. Volume для SQLite APScheduler.
6. Логирование в файл.
7. Health-команда `/ping` для мониторинга.
