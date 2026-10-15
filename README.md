## Что делаем

Бот для внешних гостей мероприятий студсовета. Один раз регистрируются, дальше — одна кнопка на каждое меро. Напоминания автоматически. Статистика доступна на основной платформе.

## Зачем

Проводится много мероприятий в год, и на каждое — новая форма и ручная работа с таблицами. Бот нужен чтобы это автоматизировать и сэкономить кучу времени - и нам и потенциальным гостям на мероприятия.

## Стек

- Python + aiogram 3.x
- PostgreSQL (общая с основным проектом)
- APScheduler + SQLite (для задач напоминаний)
- MemoryStorage — FSM aiogram (встроенный в aiogram без Redis)
- Docker, отдельный compose
- Отдельный VPS

## Схема данных

### Postgres — таблицы бота

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

### SQLite — только APScheduler
APScheduler сам генерирует таблицу apscheduler_jobs

Файл `data/scheduler.db` и volume-том в docker, чтобы не терялось при деплоях, бэкап раз в сутки (опционально).

## Права в Postgres

Создаём отдельную роль `bot_user`. Даём:
```sql
CONNECT на базу studsovet
USAGE на схему public
SELECT, INSERT, UPDATE, DELETE на guests и registrations
USAGE, SELECT на последовательности guests_id_seq и registrations_id_seq
SELECT на events — обязателен для FK-проверки и списка меро
```

На остальные таблицы (AspNetUsers, Badges) прав не даём.

## Подключение

- Сеть: VPN (WireGuard / Tailscale) или SSH-туннель
- Postgres наружу не открывать
- DATABASE_URL: `postgresql+asyncpg://bot_user:...@<vpn_ip>:5432/studsovet`

## Примерная структура бота - почитайте [курс по Aiogram](https://github.com/Digital-SGN/Handbook/tree/main/docs/Aiogram)
```text
bot/
├── handlers/
│   ├── __init__.py
│   ├── start.py          — /start, регистрация (FSM)
│   ├── events.py         — список меро, запись/отмена
│   ├── my.py             — /my, свои регистрации
│   └── admin.py          — /list, /attended
│
├── keyboards/
│   ├── __init__.py
│   ├── main.py           — главное меню
│   ├── events.py         — список меро, кнопки записи
│   └── admin.py          — админ-кнопки
│
├── middlewares/
│   ├── __init__.py
│   └── logging.py        — логирование апдейтов
│
├── db/
│   ├── __init__.py
│   ├── models.py         — SQLAlchemy: Guest, GuestRegistration
│   ├── session.py        — движок, sessionmaker
│   └── queries.py        — функции работы с БД
│
├── scheduler/
│   ├── __init__.py
│   ├── setup.py          — инициализация APScheduler + SQLite
│   └── reminders.py      — планирование и отправка напоминаний
│
├── data/
│   └── scheduler.db      — SQLite для APScheduler (volume)
│
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

**Админ** (список в .env,):

- `/list <event_id>` — кто записался
- `/attended <guest_id> <event_id>` — отметить пришедшего

## Напоминания

- При регистрации — 2 задачи в APScheduler: за 24ч и за 2ч
- Задачи в SQLite на VPS бота (volume)
- При отмене — обе задачи удаляются
- При изменении даты меро — пересоздать задачи
- misfire_grace_time=3600 — если бот лежал, догонит в течение часа
- Перед отправкой — проверить, что регистрация активна


##  На будущее
- QR-коды для отметки
- Экспорт CSV
- Mini App
- Локализация.
- И т.д.

Взаимодействие с данными будет на платформе студсовета, например сейчас у мероприятий есть поля по явке (сколько зарегистрировалось и сколько пришло), но они заполняются вручную, этот бот поможет автоматизировать это и проводить хорошую аналитику по посещаемости, также сэкономит кучу времени в будущем.

## Открытые вопросы

- VPN или SSH-туннель для связи VPS бота с Postgres
- Текст согласия на ПД и страница /privacy

## Что критично с самого начала

1. Отдельный пользователь БД с ограниченными правами
2. UNIQUE (guest_id, event_id) — защита от двойного клика
3. ON DELETE CASCADE на guests, RESTRICT на events
4. Согласие на ПД фиксируется при регистрации
5. Volume для SQLite APScheduler
6. Логирование в файл
7. Health-команда /ping для мониторинга
