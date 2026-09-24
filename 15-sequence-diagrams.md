# 15. Диаграммы последовательности

Диаграммы показывают, **как контейнеры из C4 ([раздел 12](12-c4.md)) взаимодействуют во времени** в ключевых сценариях. Особое внимание уделено транзакциям: какие шаги выполняются атомарно и где стоят ограничения БД.

| # | Сценарий | UC | Что демонстрирует |
|---|---|---|---|
| SD-1 | Запись на групповое занятие | UC-03 | Порядок проверок, атомарное занятие места, outbox |
| SD-2 | Гонка за последнее место | UC-03 | Корректность при конкурентном доступе |
| SD-3 | Бронь персонального слота | UC-04 | Блокировка тренера и проверка его занятости |
| SD-4 | Отмена записи клиентом | UC-06 | Дедлайн, атомарный возврат места, отмена напоминания |
| SD-5 | Отмена сеанса клубом и доставка push | UC-08, UC-11 | Массовая отмена в одной транзакции, работа Worker |

## SD-1. Запись на групповое занятие

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#EFF6FF','primaryBorderColor':'#2563EB','primaryTextColor':'#0F172A','lineColor':'#475569','secondaryColor':'#F8FAFC','tertiaryColor':'#FFFFFF','fontFamily':'Inter, Segoe UI, Arial, sans-serif','fontSize':'14px'}}}%%
sequenceDiagram
    autonumber
    actor C as Клиент
    participant App as Мобильное приложение
    participant API as Booking API
    participant Auth as Сервис аутентификации
    participant DB as PostgreSQL

    C->>App: Нажимает «Записаться»
    App->>API: POST /v1/bookings {session_id}<br/>Idempotency-Key: k1
    API->>Auth: Проверка токена (JWKS, кэш ключей)
    Auth-->>API: user_id
    API->>DB: Есть запись с (user_id, k1)?
    alt Повтор запроса с тем же ключом
        DB-->>API: Запись найдена
        API-->>App: 201 + та же запись (EC-02)
    else Новый запрос
        API->>DB: BEGIN
        API->>DB: SELECT сеанс
        Note over API: Проверки 2–4: не отменён,<br/>в горизонте, окно записи открыто
        API->>DB: Активная запись клиента на сеанс?
        API->>DB: Пересечение с активными записями клиента?
        alt Любая проверка не пройдена
            API->>DB: ROLLBACK + запись в BOOKING_REJECTION
            API-->>App: 409 {code: причина}
        else Проверки пройдены
            API->>DB: UPDATE session SET booked_count = booked_count + 1<br/>WHERE id = :id AND booked_count < capacity
            alt 0 строк обновлено
                API->>DB: ROLLBACK
                API-->>App: 409 SESSION_FULL
            else 1 строка обновлена
                API->>DB: INSERT booking (ACTIVE)
                API->>DB: INSERT booking_status_history
                API->>DB: INSERT notification BOOKING_CONFIRMED (PENDING)
                API->>DB: INSERT notification REMINDER (send_at = start − 2 ч)
                API->>DB: COMMIT
                API-->>App: 201 Created {booking}
                App-->>C: «Вы записаны. Отменить можно до 18:30»
            end
        end
    end
```

**Ключевые моменты**
- Занятие места, создание записи, история и уведомления фиксируются **одним COMMIT**. Частичного состояния не бывает.
- Условие `booked_count < capacity` проверяется в самом `UPDATE`. Отдельное чтение «сколько мест осталось» с последующей записью дало бы гонку (см. SD-2).
- Если параллельно проскочит вторая запись того же клиента, её отклонит уникальный индекс, а пересечение — exclusion-ограничение ([13.3](13-er-model.md#133-ограничения-целостности)). Такая ошибка БД транслируется в `ALREADY_BOOKED` или `CLIENT_TIME_CONFLICT`.
- Напоминание не создаётся, если до начала меньше 2 часов (FR-26).

## SD-2. Гонка за последнее место

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#EFF6FF','primaryBorderColor':'#2563EB','primaryTextColor':'#0F172A','lineColor':'#475569','secondaryColor':'#F8FAFC','tertiaryColor':'#FFFFFF','fontFamily':'Inter, Segoe UI, Arial, sans-serif','fontSize':'14px'}}}%%
sequenceDiagram
    autonumber
    participant A as Клиент A (API)
    participant DB as PostgreSQL
    participant B as Клиент B (API)

    Note over DB: capacity = 25, booked_count = 24
    par Одновременно
        A->>DB: UPDATE … SET booked_count = booked_count + 1<br/>WHERE id = :s AND booked_count < capacity
    and
        B->>DB: UPDATE … SET booked_count = booked_count + 1<br/>WHERE id = :s AND booked_count < capacity
    end
    Note over DB: Строка сеанса блокируется первым UPDATE.<br/>Второй ждёт COMMIT первого и перечитывает условие
    DB-->>A: 1 строка обновлена (24 → 25)
    A->>DB: INSERT booking, COMMIT
    DB-->>B: 0 строк обновлено (25 < 25 — ложь)
    B->>DB: ROLLBACK
    Note over A,B: A — 201 Created, B — 409 SESSION_FULL.<br/>booked_count = 25, овербукинга нет (EC-01)
```

Уровень изоляции — стандартный `READ COMMITTED`: в PostgreSQL при `UPDATE` конкурирующая транзакция ждёт снятия блокировки строки и заново проверяет условие `WHERE` на актуальной версии строки. Поэтому дополнительные блокировки не нужны.

## SD-3. Бронь персонального слота

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#EFF6FF','primaryBorderColor':'#2563EB','primaryTextColor':'#0F172A','lineColor':'#475569','secondaryColor':'#F8FAFC','tertiaryColor':'#FFFFFF','fontFamily':'Inter, Segoe UI, Arial, sans-serif','fontSize':'14px'}}}%%
sequenceDiagram
    autonumber
    actor C as Клиент
    participant API as Booking API
    participant DB as PostgreSQL

    C->>API: POST /v1/bookings {session_id: слот 18:00–19:00}
    API->>DB: BEGIN
    API->>DB: SELECT сеанс (type = PERSONAL)
    Note over API: Окно записи: now ≤ start − 24 ч
    API->>DB: SELECT … FROM trainer WHERE id = :t FOR UPDATE
    Note over DB: Брони этого тренера выполняются<br/>строго по очереди (ADR-06)
    API->>DB: Есть ли у тренера пересечение с [18:00, 19:00)?<br/>групповой SCHEDULED-сеанс или слот с ACTIVE-записью
    alt Тренер занят
        API->>DB: ROLLBACK
        API-->>C: 409 TRAINER_BUSY
    else Тренер свободен
        API->>DB: UPDATE session … WHERE booked_count < capacity (= 1)
        API->>DB: INSERT booking, history, notifications
        API->>DB: COMMIT
        API-->>C: 201 Created
        Note over C: Цену и оплату клиент<br/>согласует с тренером напрямую
    end
```

Блокировка строки тренера нужна потому, что два клиента могут одновременно бронировать **разные**, но пересекающиеся слоты одного тренера (18:00–19:00 и 18:30–19:30). Условный `UPDATE` защищает только один сеанс, а блокировка сериализует все брони тренера (EC-12).

## SD-4. Отмена записи клиентом

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#EFF6FF','primaryBorderColor':'#2563EB','primaryTextColor':'#0F172A','lineColor':'#475569','secondaryColor':'#F8FAFC','tertiaryColor':'#FFFFFF','fontFamily':'Inter, Segoe UI, Arial, sans-serif','fontSize':'14px'}}}%%
sequenceDiagram
    autonumber
    actor C as Клиент
    participant App as Мобильное приложение
    participant API as Booking API
    participant DB as PostgreSQL

    C->>App: «Отменить» → подтверждает
    App->>API: POST /v1/bookings/{id}/cancel
    API->>DB: BEGIN
    API->>DB: SELECT booking WHERE id = :id AND user_id = :u FOR UPDATE
    alt Не найдена или чужая
        API-->>App: 404 (EC-18)
    else Уже CANCELLED_BY_CLIENT
        API-->>App: 200 + текущее состояние (EC-16)
    else COMPLETED или CANCELLED_BY_CLUB
        API-->>App: 409 BOOKING_NOT_ACTIVE
    else ACTIVE
        Note over API: Дедлайн = start − 30 мин (групповое)<br/>или start − 24 ч (персональное).<br/>free_cancel = true → до начала
        alt now > дедлайна
            API->>DB: ROLLBACK
            API-->>App: 409 CANCELLATION_DEADLINE_PASSED
        else Отмена разрешена
            API->>DB: UPDATE booking SET status = CANCELLED_BY_CLIENT
            API->>DB: UPDATE session SET booked_count = booked_count − 1
            API->>DB: UPDATE notification SET push_status = CANCELLED<br/>WHERE booking_id = :id AND type = REMINDER
            API->>DB: INSERT history, notification BOOKING_CANCELLED
            API->>DB: COMMIT
            API-->>App: 200 {status: CANCELLED_BY_CLIENT}
            App-->>C: Запись в «Истории», место свободно
        end
    end
```

`FOR UPDATE` на записи защищает от двойной отмены параллельными запросами: второй запрос дождётся первого, увидит статус `CANCELLED_BY_CLIENT` и не уменьшит счётчик повторно.

## SD-5. Отмена сеанса клубом и доставка push

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#EFF6FF','primaryBorderColor':'#2563EB','primaryTextColor':'#0F172A','lineColor':'#475569','secondaryColor':'#F8FAFC','tertiaryColor':'#FFFFFF','fontFamily':'Inter, Segoe UI, Arial, sans-serif','fontSize':'14px'}}}%%
sequenceDiagram
    autonumber
    actor T as Тех. специалист
    participant API as Booking API
    participant DB as PostgreSQL
    participant W as Worker
    participant P as APNs / FCM
    participant App as Приложение клиента

    T->>API: POST /internal/v1/sessions/{id}/cancel {reason}
    API->>DB: BEGIN
    API->>DB: UPDATE session SET status = CANCELLED<br/>WHERE id = :id AND status = SCHEDULED
    API->>DB: UPDATE booking SET status = CANCELLED_BY_CLUB<br/>WHERE session_id = :id AND status = ACTIVE RETURNING id, user_id
    API->>DB: UPDATE session SET booked_count = 0
    API->>DB: Отменить REMINDER по этим записям
    API->>DB: INSERT history × N, notification SESSION_CANCELLED × N
    API->>DB: COMMIT
    API-->>T: 200 {affected_bookings: N}

    loop Каждые несколько секунд
        W->>DB: SELECT notification WHERE push_status = PENDING<br/>AND send_at ≤ now FOR UPDATE SKIP LOCKED LIMIT 100
        W->>P: Отправка push на устройства клиента
        alt Доставлено
            W->>DB: push_status = SENT
        else Ошибка
            W->>DB: attempts + 1, повтор с задержкой (до 3 раз), затем FAILED
        else Токен недействителен
            W->>DB: DELETE device
        end
    end
    P-->>App: «Занятие отменено клубом»
```

`FOR UPDATE SKIP LOCKED` позволяет запускать несколько экземпляров Worker: каждый забирает свою пачку уведомлений, и ни одно не отправляется дважды. Тот же обработчик отправляет напоминания: это уведомления `REMINDER`, у которых наступило `send_at`.
