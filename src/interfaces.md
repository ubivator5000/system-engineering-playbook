## 10. Интерфейсы

---

#### Контракт №1: Создание платежа (от WordPress/Tilda)

| Параметр | Значение |
|----------|----------|
| **Endpoint** | `POST /api/v1/payment` |
| **Версия** | v1 |
| **Протокол** | HTTPS, синхронный |
| **Аутентификация** | HMAC-SHA256 (заголовки: `X-Public-Key`, `X-Signature`, `X-Timestamp`) |
| **Идемпотентность** | Заголовок `Idempotency-Key` (обязательный) |

**Заголовки запроса:**

| Заголовок | Тип | Обязательность | Описание |
|-----------|-----|----------------|----------|
| `X-Public-Key` | string | ✅ Да | Публичный ключ школы |
| `X-Signature` | string | ✅ Да | HMAC-SHA256 подпись |
| `X-Timestamp` | int64 | ✅ Да | Unix timestamp (отклонение >5 мин → ошибка) |
| `Idempotency-Key` | string | ✅ Да | Уникальный ключ идемпотентности (макс 128 символов) |
| `Content-Type` | string | ✅ Да | `application/json` |

**Тело запроса (Request Body):**

| Поле | Тип | Обязательность | Описание |
|------|-----|----------------|----------|
| `amount` | decimal(10,2) | ✅ Да | Сумма платежа |
| `product_id` | string(36) | ✅ Да | ID курса (UUID) |
| `product_name` | string(255) | ✅ Да | Название курса |
| `student_name` | string(100) | ✅ Да | Имя студента |
| `student_surname` | string(100) | ✅ Да | Фамилия студента |
| `student_patronymic` | string(100) | ❌ Нет | Отчество |
| `student_phone` | string(20) | ✅ Да | Телефон (формат: +7XXXXXXXXXX) |
| `is_subscription` | boolean | ❌ Нет (default: false) | Рекуррентный платёж |
| `return_url` | string(255) | ❌ Нет | URL для редиректа после оплаты |

**Пример запроса:**

```json
{
    "amount": 5000.00,
    "product_id": "550e8400-e29b-41d4-a716-446655440000",
    "product_name": "Python для начинающих",
    "student_name": "Иван",
    "student_surname": "Иванов",
    "student_phone": "+79161234567",
    "is_subscription": true
}

**Ответы (Responses):**

| HTTP код | Ситуация | Тело ответа |
|----------|----------|-------------|
| 200 OK | Платёж создан | `{"payment_id": "uuid", "redirect_url": "https://tbank.ru/pay/xxx", "code": 200}` |
| 400 Bad Request | Невалидные поля | `{"error": "field amount is required", "code": 400}` |
| 401 Unauthorized | Неверная подпись | `{"error": "invalid signature", "code": 401}` |
| 401 Unauthorized | Timestamp старше 5 минут | `{"error": "timestamp expired", "code": 401}` |
| 409 Conflict | Idempotency-Key уже использован | `{"payment_id": "uuid", "status": "existing", "code": 409}` |
| 429 Too Many Requests | Превышен rate limit (5 RPM) | `{"error": "rate limit exceeded", "retry_after": 60, "code": 429}` |
| 500 Internal Error | Внутренняя ошибка сервера | `{"error": "internal server error", "code": 500}` |
| 503 Service Unavailable | Т-Банк недоступен | `{"error": "bank service unavailable", "code": 503}` |

**Версионирование:**

- Текущая версия: `v1` (URL: `/api/v1/payment`)
- При изменениях — новая версия: `/api/v2/payment`
- Старая версия поддерживается минимум 6 месяцев

**Ретраи для ошибок:**

| HTTP код | Retry strategy |
|----------|----------------|
| 429, 500, 503 | Повтор через 1, 2, 4, 8 секунд (exponential backoff) |
| 400, 401, 409 | Без ретрая (ошибка клиента) |

# Контракт №2: Вебхук от Т-Банка

## Параметры

| Параметр | Значение |
|----------|----------|
| **Endpoint** | `POST /api/v1/webhook/tbank` |
| **Версия** | v1 |
| **Протокол** | HTTPS, синхронный |
| **Аутентификация** | Проверка IP белого списка + подпись от Т-Банка |

---

## Заголовки запроса

| Заголовок | Тип | Обязательность | Описание |
|-----------|------|----------------|-----------|
| `X-TBank-Signature` | `string` | ✅ Да | Подпись вебхука от Т-Банка |
| `Content-Type` | `string` | ✅ Да | `application/json` |

---

## Тело запроса (Request Body)

| Поле | Тип | Обязательность | Описание |
|------|------|----------------|-----------|
| `payment_id` | `string(36)` | ✅ Да | ID платежа в системе Т-Банка |
| `order_id` | `string(36)` | ✅ Да | ID платежа в PayLect (соответствует `payment_id`) |
| `status` | `enum` | ✅ Да | `succeeded` / `failed` / `pending` / `refunded` |
| `amount` | `decimal(10,2)` | ✅ Да | Сумма списания |
| `card_token` | `string` | ❌ Нет | Токен карты (для рекуррентных платежей) |
| `error_code` | `string` | ❌ Нет | Код ошибки (при `status=failed`) |
| `error_message` | `string` | ❌ Нет | Описание ошибки |
| `timestamp` | `int64` | ✅ Да | Unix timestamp |

---

## Примеры запросов

### ✅ Успешный платёж

```json
{
  "payment_id": "tbank_123456",
  "order_id": "550e8400-e29b-41d4-a716-446655440001",
  "status": "succeeded",
  "amount": 5000.00,
  "card_token": "tok_abcdef123456",
  "timestamp": 1700000000
}

### ❌ Отказ карты

```json
{
  "payment_id": "tbank_123457",
  "order_id": "550e8400-e29b-41d4-a716-446655440002",
  "status": "failed",
  "amount": 5000.00,
  "error_code": "insufficient_funds",
  "error_message": "Недостаточно средств на карте",
  "timestamp": 1700000000
}

## Ответы (Responses)

| HTTP код | Ситуация | Тело ответа |
|----------|----------|-------------|
| **200 OK** | Вебхук успешно обработан | `{"status": "ok", "code": 200}` |
| **500 Internal Error** | Ошибка обработки (банк повторит) | `{"error": "processing failed", "code": 500}` |

## Версионирование

- Текущая версия: `v1` (URL: `/api/v1/webhook/tbank`)
- При изменениях — новая версия: `/api/v2/webhook/tbank`
- Старая версия поддерживается минимум 6 месяцев

## Ретраи для ошибок

| HTTP код | Retry strategy |
|----------|----------------|
| 429, 500, 503 | Повтор через 1, 2, 4, 8 секунд (exponential backoff) |
| 400, 401, 409 | Без ретрая (ошибка клиента) |

#### Контракт №3: Выгрузка платежей в 1С (и для внешних отчётов)

| Параметр | Значение |
|----------|----------|
| **Endpoint** | `GET /api/v1/payments` |
| **Версия** | v1 |
| **Протокол** | HTTPS, синхронный |
| **Аутентификация** | API-ключ (заголовок `X-API-Key`) |

**Заголовки запроса:**

| Заголовок | Тип | Обязательность | Описание |
|-----------|-----|----------------|----------|
| `X-API-Key` | string | ✅ Да | API-ключ школы (из личного кабинета PayLect) |
| `Content-Type` | string | ✅ Да | `application/json` |

**Параметры запроса (Query parameters):**

| Параметр | Тип | Обязательность | Описание | Пример |
|----------|-----|----------------|----------|--------|
| `from` | date | ✅ Да | Начало периода (YYYY-MM-DD) | `2026-01-01` |
| `to` | date | ✅ Да | Конец периода (YYYY-MM-DD) | `2026-12-31` |
| `status` | string | ❌ Нет | Фильтр по статусу (`succeeded` / `failed` / `refunded`) | `succeeded` |
| `limit` | int | ❌ Нет (default: 100, max: 1000) | Лимит записей | `50` |
| `offset` | int | ❌ Нет (default: 0) | Смещение для пагинации | `100` |

**Пример запроса:**

```http
GET /api/v1/payments?from=2026-01-01&to=2026-12-31&status=succeeded&limit=100&offset=0

## Пример ответа (200 OK)

```json
{
  "total": 150,
  "offset": 0,
  "limit": 100,
  "payments": [
    {
      "payment_id": "550e8400-e29b-41d4-a716-446655440001",
      "date": "2026-05-06T10:30:00Z",
      "amount": 5000.00,
      "student_name": "Иван",
      "student_surname": "Иванов",
      "student_patronymic": null,
      "student_phone": "+79161234567",
      "course_id": "550e8400-e29b-41d4-a716-446655440000",
      "course_name": "Python для начинающих",
      "status": "succeeded",
      "is_subscription": true,
      "subscription_id": "550e8400-e29b-41d4-a716-446655440003"
    }
  ]
}

## Ответы (Responses)

| HTTP код | Ситуация | Тело ответа |
|----------|----------|-------------|
| **200 OK** | Успешно | `{"total": 150, "payments": [...]}` |
| **400 Bad Request** | Неверный период (from > to) | `{"error": "invalid date range", "code": 400}` |
| **400 Bad Request** | Неверный статус | `{"error": "invalid status value", "code": 400}` |
| **401 Unauthorized** | Неверный или отсутствующий API-ключ | `{"error": "invalid api key", "code": 401}` |
| **429 Too Many Requests** | Превышен rate limit (10 запросов/мин на ключ) | `{"error": "rate limit exceeded", "retry_after": 60, "code": 429}` |
| **500 Internal Error** | Внутренняя ошибка сервера | `{"error": "internal server error", "code": 500}` |

## Версионирование

- Текущая версия: `v1` (URL: `/api/v1/payments`)
- При изменениях — новая версия: `/api/v2/payments`
- Старая версия поддерживается минимум 6 месяцев

## Ретраи для ошибок

| HTTP код | Retry strategy |
|----------|----------------|
| 429, 500, 503 | Повтор через 1, 2, 4, 8 секунд (exponential backoff) |
| 400, 401 | Без ретрая (ошибка клиента) |