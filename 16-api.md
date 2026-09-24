# 16. API

Полная спецификация — [`api/openapi.yaml`](/openapi.yaml) (OpenAPI 3.0.3). Её можно открыть в [Swagger Editor](https://editor.swagger.io/): вставьте содержимое файла, и слева появится интерактивная документация. Этот раздел объясняет принципы API и ключевые решения.

## 16.1. Эндпоинты

| Метод | Путь | Назначение | FR | US |
|---|---|---|---|---|
| GET | `/v1/schedule` | Расписание на день с фильтрами | FR-04…FR-07 | US-01…US-03 |
| GET | `/v1/sessions/{sessionId}` | Карточка сеанса | FR-08 | US-04 |
| POST | `/v1/bookings` | Записаться на сеанс | FR-09…FR-15 | US-05…US-10 |
| GET | `/v1/bookings/{bookingId}` | Карточка записи | FR-03 | US-13 |
| POST | `/v1/bookings/{bookingId}/cancel` | Отменить запись | FR-16…FR-20 | US-11, US-12 |
| GET | `/v1/me/bookings?scope=upcoming\|history` | Мои записи | FR-03 | US-13 |
| GET | `/v1/me/notifications` | Лента уведомлений и счётчик непрочитанных | FR-29 | US-14 |
| POST | `/v1/me/notifications/{id}/read` | Отметить прочитанным | FR-29 | US-14 |
| POST | `/v1/me/devices` | Зарегистрировать токен устройства для push | FR-29 | US-14 |
| POST | `/internal/v1/sessions/{sessionId}/cancel` | Отменить сеанс (операция клуба) | FR-22 | US-16 |
| POST | `/internal/v1/sessions/{sessionId}/trainer` | Заменить тренера группового сеанса | FR-23 | US-17 |

Отмена записи — отдельное действие `POST …/cancel`, а не `DELETE /bookings/{id}`. Запись не удаляется: она меняет статус и остаётся в истории клиента и в отчётах (FR-31).

## 16.2. Соглашения

| Тема | Правило |
|---|---|
| Аутентификация | `Authorization: Bearer <token>` от внешнего сервиса. `user_id` берётся только из токена, а не из тела запроса. Служебный API — отдельный сервисный токен |
| Время | UTC, ISO 8601 (`2026-10-05T16:00:00Z`). Приложение показывает время клуба |
| Идемпотентность | `POST /v1/bookings` требует заголовок `Idempotency-Key` (UUID, генерирует приложение на каждую попытку записи). Повтор с тем же ключом в течение 24 ч возвращает исходный ответ. Отмена идемпотентна по смыслу: повторная отмена возвращает текущее состояние |
| Пагинация | Курсорная: `cursor` + `limit` (по умолчанию 20, максимум 100), в ответе `next_cursor` |
| Ошибки | Единый формат `{code, message, details}`. `code` — машиночитаемая причина, `message` — текст для клиента из [таблицы 6.5](06-user-requirements.md#65-сообщения-пользователю) |
| Чужие объекты | Запрос к записи другого клиента (просмотр, отмена) получает `404`, а не `403`: так API не подтверждает, что запись с таким id существует (FR-03). Код `409` здесь не применяется: он означает конфликт с состоянием ресурса (мест нет, клиент уже записан) |
| Лимиты | 30 запросов/мин на запись и отмену от пользователя, сверх лимита — `429` с `Retry-After` (NFR-12) |

## 16.3. Коды ошибок

| Код | HTTP | Когда | Где проверяется |
|---|---|---|---|
| `SESSION_NOT_FOUND` | 404 | Сеанса нет | FR-09, шаг 1 |
| `SESSION_CANCELLED` | 409 | Сеанс отменён клубом | FR-09, шаг 2 |
| `OUT_OF_HORIZON` | 409 / 400 | Дальше 14 дней (при записи — 409, при запросе расписания — 400) | FR-04, FR-09, шаг 3 |
| `BOOKING_CLOSED` | 409 | Групповое уже началось, до персонального меньше 24 ч | FR-09, шаг 4 |
| `ALREADY_BOOKED` | 409 | У клиента уже есть активная запись на сеанс | FR-09, шаг 5; уникальный индекс |
| `CLIENT_TIME_CONFLICT` | 409 | Пересечение с другой активной записью клиента, в `details` — её id | FR-09, шаг 6; exclusion-ограничение |
| `TRAINER_BUSY` | 409 | У тренера пересекающееся занятие | FR-09, шаг 7; FR-23 |
| `SESSION_FULL` | 409 | Мест нет | FR-09, шаг 8 |
| `CANCELLATION_DEADLINE_PASSED` | 409 | Срок отмены прошёл | FR-16 |
| `BOOKING_NOT_ACTIVE` | 409 | Запись завершена или отменена клубом | FR-19 |
| `SESSION_NOT_SCHEDULED` | 409 | Операция клуба над отменённым или прошедшим сеансом | FR-22, FR-23 |
| `REPLACEMENT_NOT_ALLOWED` | 409 | Попытка заменить тренера персонального слота | FR-23 |
| `VALIDATION_ERROR` | 400 | Неверные параметры запроса: нет обязательного поля, неверный формат даты или id | Все эндпоинты |
| `UNAUTHORIZED` | 401 | Токен отсутствует, невалиден или просрочен | FR-01, EC-29 |
| `NOT_FOUND` | 404 | Объект не найден или принадлежит другому клиенту | FR-03, EC-18 |
| `RATE_LIMITED` | 429 | Превышен лимит запросов на запись и отмену | NFR-12, EC-31 |
| `SERVICE_UNAVAILABLE` | 503 | Недоступен сервис аутентификации или БД | FR-01, EC-30 |

## 16.4. Пример: запись на групповое занятие

**Запрос**

```http
POST /v1/bookings HTTP/1.1
Authorization: Bearer eyJhbGciOi...
Idempotency-Key: 5b7c1f0e-2a51-4a8e-9d1c-0f3a7e2b9c11
Content-Type: application/json

{ "session_id": "7f3d2c1b-9a8e-4f6d-8c7b-6a5e4d3c2b1a" }
```

**Успешный ответ**

```http
HTTP/1.1 201 Created
Content-Type: application/json

{
  "id": "a1b2c3d4-e5f6-4a7b-8c9d-0e1f2a3b4c5d",
  "status": "ACTIVE",
  "free_cancel": false,
  "cancel_deadline": "2026-10-05T15:30:00Z",
  "created_at": "2026-10-03T08:12:45Z",
  "session": {
    "id": "7f3d2c1b-9a8e-4f6d-8c7b-6a5e4d3c2b1a",
    "type": "GROUP",
    "activity": { "id": "…", "name": "Йога", "level": "ALL", "duration_min": 60 },
    "trainer": { "id": "…", "full_name": "Иванов Сергей" },
    "hall": { "id": "…", "name": "Зал 1" },
    "starts_at": "2026-10-05T16:00:00Z",
    "ends_at": "2026-10-05T17:00:00Z",
    "capacity": 25,
    "available_seats": 6,
    "status": "SCHEDULED",
    "booking_state": "BOOKED_BY_ME"
  }
}
```

**Отказ**

```http
HTTP/1.1 409 Conflict
Content-Type: application/json

{ "code": "SESSION_FULL", "message": "Мест нет — все места уже заняты. Посмотрите это занятие в другие дни" }
```

## 16.5. Атомарность на уровне SQL

Это ответ на пункт ТЗ «возврат количества мест должен быть атомарным (конкурентный доступ)». Сценарии во времени показаны в [SD-1…SD-4](15-sequence-diagrams.md).

**Занять место (внутри транзакции создания записи)**

```sql
UPDATE session
   SET booked_count = booked_count + 1
 WHERE id = :session_id
   AND status = 'SCHEDULED'
   AND booked_count < capacity
RETURNING booked_count;
-- 0 строк → SESSION_FULL, транзакция откатывается
-- 1 строка → INSERT booking, history, notifications; COMMIT
```

**Вернуть место при отмене клиентом**

```sql
-- запись уже заблокирована: SELECT … FROM booking WHERE id = :id AND user_id = :u FOR UPDATE
UPDATE booking
   SET status = 'CANCELLED_BY_CLIENT', updated_at = now()
 WHERE id = :booking_id AND status = 'ACTIVE';

UPDATE session
   SET booked_count = booked_count - 1
 WHERE id = :session_id;
-- CHECK (booked_count BETWEEN 0 AND capacity) не даст уйти в минус
```

**Сериализация броней персонального тренера (ADR-06)**

```sql
SELECT id FROM trainer WHERE id = :trainer_id FOR UPDATE;

SELECT 1
  FROM session s
 WHERE s.trainer_id = :trainer_id
   AND s.status = 'SCHEDULED'
   AND s.id <> :session_id
   AND tstzrange(s.starts_at, s.ends_at) && tstzrange(:starts_at, :ends_at)
   AND (s.type = 'GROUP' OR s.booked_count > 0)
 LIMIT 1;
-- найдено → TRAINER_BUSY
```

**Ограничения, которые страхуют код**

```sql
ALTER TABLE session
  ADD CONSTRAINT seats_within_capacity CHECK (booked_count BETWEEN 0 AND capacity),
  ADD CONSTRAINT personal_capacity_one CHECK (type <> 'PERSONAL' OR capacity = 1);

CREATE UNIQUE INDEX booking_one_active_per_session
  ON booking (session_id, user_id) WHERE status = 'ACTIVE';

CREATE EXTENSION IF NOT EXISTS btree_gist;
ALTER TABLE booking
  ADD CONSTRAINT booking_no_client_overlap
  EXCLUDE USING gist (user_id WITH =, tstzrange(starts_at, ends_at, '[)') WITH &&)
  WHERE (status = 'ACTIVE');

CREATE UNIQUE INDEX booking_idempotency
  ON booking (user_id, idempotency_key);
```
