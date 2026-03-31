# Цифровая экосистема частной клиники

[![Python](https://img.shields.io/badge/Python-3.11-blue?logo=python)](https://www.python.org/)
[![FastAPI](https://img.shields.io/badge/FastAPI-0.100+-009688?logo=fastapi)](https://fastapi.tiangolo.com/)
[![React](https://img.shields.io/badge/React-18-blue?logo=react)](https://react.dev/)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-15-blue?logo=postgresql)](https://www.postgresql.org/)
[![Telegram API](https://img.shields.io/badge/Telegram_Bot_API-blue?logo=telegram)](https://core.telegram.org/bots/api)

**Статус:** Production с июля 2024  
**Роль:** Системный аналитик — полный цикл  
**Код:** закрыт по соображениям безопасности данных пациентов (NDA), репозиторий содержит архитектурную документацию и демонстрацию интерфейсов.

---

## Интерфейсы системы

### Интерфейс пациента (Telegram Bot)
*Выбор услуги, врача и свободного времени. Быстрая запись без звонков в регистратуру.*
<p align="center">
  <img width="280" alt="Telegram Bot Step 1" src="https://github.com/user-attachments/assets/f0a67b16-0753-4fd0-ae47-89023db62d45" />
  &nbsp;&nbsp;&nbsp;&nbsp;
  <img width="280" alt="Telegram Bot Step 2" src="https://github.com/user-attachments/assets/c77ce379-8cb5-4263-8074-2a0298d10e50" />
</p>

### Рабочий стол врача/администратора (React SPA)
*Мониторинг записей, управление статусами и карточками пациентов.*
<p align="center">
  <img width="100%" alt="Web Panel Dashboard" src="https://github.com/user-attachments/assets/bac91cd4-b5d1-4878-944c-2334362cd55d" />
  <br><br>
  <img width="100%" alt="Web Panel UI 2" src="https://github.com/user-attachments/assets/f03c4f7b-4f37-42ac-8bdb-b052c0f427f4" />
  <br><br>
  <img width="100%" alt="Web Panel UI 3" src="https://github.com/user-attachments/assets/8e87b2f6-3f35-4576-9b2c-27dfd78cd612" />
</p>

### Сетка расписания
*Единая точка управления слотами с мгновенной синхронизацией. Исключение двойных записей.*
<p align="center">
  <img width="100%" alt="Schedule Grid" src="https://github.com/user-attachments/assets/a4cdabad-fc24-4e52-be90-7728d777d213" />
</p>

---

## Задача
До внедрения системы клиника сталкивалась с проблемами дублирования записей, ручного ведения расписания и потери заявок из-за разрозненности каналов коммуникации. 
Задачей было спроектировать и запустить отказоустойчивую централизованную экосистему, объединяющую Telegram-бота для пациентов и веб-панель для врачей с единой точкой правды в базе данных, а также оптимизировать производительность интерфейсов.

---

## Архитектура (C4 Context)

Ключевое решение: полная синхронизация двух каналов (Telegram Bot и React Web Panel) осуществляется строго через единые REST API эндпоинты на FastAPI. Это архитектурное ограничение исключает гонку данных при одновременной записи и обеспечивает мгновенное обновление интерфейсов для всех участников процесса.

```mermaid
flowchart TB
    Patient(["👨‍🦰 Пациент"])
    Doctor(["👨‍⚕️ Врач / Администратор"])

    subgraph System ["Медицинская информационная система"]
        direction TB
        TG["🤖 Telegram Bot<br/>(Интерфейс пациента)"]
        Web["💻 React Web Panel<br/>(SPA панель врача)"]
        API["⚙️ FastAPI Backend<br/>(Единое REST API ядро)"]
        DB[("🗄️ PostgreSQL<br/>(Единая точка правды)")]
    end

    Patient -->|Бронирует слоты| TG
    Doctor -->|Управляет расписанием| Web
    TG -->|Запросы JSON / REST API| API
    Web -->|Запросы JSON / REST API| API
    API -->|Транзакции ACID| DB
```

---

## Процесс записи (Предотвращение коллизий)

Процесс спроектирован с учетом предотвращения коллизий. Блокировка слота и создание записи происходят в рамках одной транзакции базы данных (ACID), что делает физически невозможным дублирование записи на одно и то же время разными пациентами.

```mermaid
flowchart TD
    A([Пациент открывает бота]) --> B[Выбор врача и даты]
    B --> C[Бот запрашивает доступные слоты]
    C --> D{FastAPI: Слот свободен?}
    
    D -- Да --> E[Начало транзакции БД]
    E --> F[Блокировка слота в PostgreSQL]
    F --> G[Создание записи в 'appointments']
    G --> H[COMMIT транзакции]
    H --> I([Уведомление пациенту об успехе])
    I --> J([Врач видит запись в веб-панели])
    
    D -- Нет --> K[Откат транзакции / ROLLBACK]
    K --> L[Уведомление: Слот занят]
    L --> B
```

---

## Информационная модель (ERD)

База данных спроектирована по правилам нормализации. Ниже приведена схема основных сущностей, задействованных в процессе записи и управления расписанием (реальные поля из production БД).

```mermaid
erDiagram
    telegram_users {
        int id PK
        varchar username
        varchar telegram_id
        boolean is_active
        timestamp last_login
    }
    
    patients {
        int id PK
        bigint user_id FK "Telegram ID"
        varchar full_name
        date birth_date
        varchar phone_number
    }
    
    doctors {
        int id PK
        varchar full_name
        varchar department
        varchar specialization
        boolean is_active
    }
    
    schedules {
        int id PK
        int doctor_id FK
        date date
        jsonb time_slots
    }
    
    appointments {
        int id PK
        int user_id FK
        int doctor_id FK
        timestamp date_time
        varchar status "scheduled, confirmed, canceled"
        text notes
    }

    telegram_users ||--o{ patients : "auth"
    patients ||--o{ appointments : "books"
    doctors ||--o{ appointments : "receives"
    doctors ||--o{ schedules : "has"
```

---

## Результаты
* **Оптимизация производительности:** Показатель **Lighthouse Score вырос с 65 до 95+** за счет правильной конфигурации SPA (React + Vite) и кэширования статики на уровне Nginx.
* **Стабильность:** Система успешно введена в эксплуатацию и работает у заказчика без замечаний с июля 2024 года.

---

## Технологический стек
* **Backend:** Python, FastAPI, SQLAlchemy
* **Frontend:** React 18, Vite, Tailwind CSS
* **Database:** PostgreSQL
* **Integration:** Telegram Bot API (aiogram)
```
