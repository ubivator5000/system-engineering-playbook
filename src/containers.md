### 8.1. Контейнеры (C4 Level 2)

| № | Контейнер | Тип | Обязанность | Что хранит | Технологии / Реализация |
|---|-----------|-----|-------------|------------|------------------------|
| 1 | **API Gateway** | Сервис | Аутентификация (HMAC), rate limiting (5 rpm/IP), приём запросов от WordPress/Tilda/1С, проверка Idempotency-Key | stateless | Go (net/http) + Redis (rate limiting counters) |
| 2 | **Payment Core** | Сервис | Ядро платежей: вызов Т-Банка, рекуррентные списания, обработка вебхуков, parent–child связь (подписки) | stateless | Go + PostgreSQL (транзакции) |
| 3 | **Payment Queue** | Очередь | Буфер для входящих запросов на оплату, защита от всплесков, dead-letter, at least once | persistent (сообщения) | RabbitMQ (persistent queues) / Apache Kafka |
| 4 | **Retry Scheduler** | Сервис | Повторные списания (1ч/24ч/72ч), напоминания за 3 дня до списания, отмена retry при успехе | stateful (таблица scheduled_jobs) | Go + leader election (PostgreSQL advisory lock / K8s lease) |
| 5 | **School Cabinet** | Сервис | Личный кабинет школы:<br>• Регистрация школы<br>• Генерация API-ключей (Public/Secret)<br>• Ролевая модель (CEO/менеджер/бухгалтер/админ)<br>• Настройки retry (0–5)<br>• Статусы интеграций<br>• Отображение дашбордов (читает из Dashboard API)<br>• Просмотр списка курсов, студентов, платежей (читает из PostgreSQL) | stateless (сессии в Redis) | Go / Python (Django/FastAPI) + JWT + PostgreSQL + Dashboard API |
| 6 | **Analytics Engine** | Сервис | Сбор событий из очереди, расчёт агрегатов:<br>• Горячие метрики (today/week) → Redis<br>• Дневные агрегаты → PostgreSQL<br>• LTV / отток / MRR → пересчёт раз в час/день | stateless | Go + Python (pandas) |
| 7 | **Dashboard API** | Сервис | Отдача метрик в School Cabinet:<br>• Горячие (today/week/month) → из Redis (<1 мс)<br>• Исторические (квартал/год/3 года) → из PostgreSQL (10–50 мс)<br>• Экспорт CSV/JSON | stateless | Go + Redis + PostgreSQL |
| 8 | **Notification Dispatcher** | Сервис | Отправка уведомлений: менеджеру о покупке (ВК), студенту о напоминании (ВК/СМС), менеджеру о 3 отказах, fallback | stateless | Go + API VK + API СМС + RabbitMQ |
| 9 | **PostgreSQL** | База данных | Транзакционные данные:<br>• Платежи, подписки, школы, API-ключи<br>• Idempotency-Key (30 дней)<br>• Аудит-логи (append only, SHA256)<br><br>Агрегатные таблицы:<br>• course_daily_revenue<br>• course_monthly_revenue<br>• ltv_by_student, churn_monthly | stateful (диск) | PostgreSQL 15+ + реплика |
| 10 | **Redis** | Кэш / In-memory БД | Горячие метрики (TTL: today=24h, week=7d, month=30d):<br>• Выручка по курсам и школам<br>• LTV по курсам<br>• Количество активных подписок<br><br>Инфраструктура:<br>• Сессии School Cabinet (TTL 24h)<br>• Кэш API-ключей (TTL 5 мин)<br>• Rate limiting counters (TTL 1 мин) | stateful (RAM + disk snapshot) | Redis 7+ + persistence (RDB/AOF) + sentinel |

---

### 8.2. Схема контейнеров

@drawio{https://github.com/ubivator5000/system-engineering-playbook/blob/main/src/diagrams/C2.drawio}