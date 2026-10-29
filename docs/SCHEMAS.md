# Схемы проекта

Это проектные схемы редакции 3, не отражение уже созданной БД. Правила и ограничения: [SPECIFICATION.md](SPECIFICATION.md). SVG можно открыть без Mermaid. Исходники для редактирования: [classes.dot](diagrams/classes.dot), [er.dot](diagrams/er.dot). Схема показывает основные связи; полный набор UNIQUE/CHECK/FK перечислен в спецификации и будет закреплён SQL-миграциями.

## 1. Архитектура

```mermaid
flowchart LR
    UI[HTML / CSS / JavaScript] --> C[Controller + DTO]
    C --> S[Service: права и транзакции]
    S --> D[Domain: состояние и правила]
    S --> R[Repository]
    R --> DB[(PostgreSQL)]
    S --> G[PaymentGateway: вызов вне транзакции заказа]
    G --> SIM[Симулятор: отдельная транзакция]
    SIM --> DB
```

## 2. Основные классы

![Диаграмма классов](diagrams/classes.svg)

Стрелка композиции означает владение составными частями. Билет относится к пассажиру, а заказ — к аккаунту покупателя; это разные связи. `Money` используется всеми денежными объектами. `RefundPolicy` содержит две учебные реализации, демонстрируя полиморфизм без искусственного наследования сущностей.

```mermaid
classDiagram
    class Booking {
        UUID id
        BookingStatus status
        Instant holdUntil
        Money total
        confirm()
        expire()
        cancelPassengers()
    }
    class BookingPassenger {
        LocalDate birthDate
        PassengerStatus status
        cancel()
        expire()
    }
    class FlightSeat {
        SeatStatus status
        Instant holdUntil
        hold()
        book()
        release()
    }
    class PaymentGateway {
        <<interface>>
        charge()
        findCharge()
        refund()
        findRefund()
    }
    class RefundPolicy {
        <<interface>>
        calculateFareRefund()
    }
    User "1" --> "0..*" Booking : owns
    Flight "1" --> "0..*" Booking
    Flight "1" *-- "1..*" FlightSeat
    Aircraft "1" *-- "1..*" AircraftSeat
    Flight "0..*" --> "1" Aircraft
    Flight "1" *-- "1..*" FlightFare
    Booking "1" *-- "1..9" BookingPassenger
    BookingPassenger "0..*" --> "1" FlightSeat : historical assignment
    BookingPassenger "1" *-- "3" BookingPriceLine
    Booking "1" --> "0..*" Payment
    Payment "1" --> "0..*" Refund
    BookingPassenger "1" --> "0..1" Ticket
    BookingPassenger "1" --> "0..*" CheckIn
    User "1" --> "0..*" Notification
    PaymentService ..> PaymentGateway
    PaymentGateway <|.. SimulatedPaymentGateway
    RefundPolicy <|.. BasicRefundPolicy
    RefundPolicy <|.. FlexRefundPolicy
    PricingService ..> Money
    RefundService ..> RefundPolicy
```

## 3. ER-схема

![Связи таблиц](diagrams/er.svg)

Все основные PK — UUID. Денежные поля numeric(12,2), время timestamptz, даты рождения date, enum хранится текстом с CHECK. Служебные timestamps и индексы запросов на схеме опущены для читаемости. Составные ключи раздела 16 спецификации обязательны, даже когда на рисунке связь показана одной стрелкой.

```mermaid
erDiagram
    USERS ||--o{ BOOKINGS : owns
    AIRPORTS ||--o{ FLIGHTS : origin
    AIRPORTS ||--o{ FLIGHTS : destination
    AIRCRAFT ||--|{ AIRCRAFT_SEATS : template
    AIRCRAFT ||--o{ FLIGHTS : operates
    FLIGHTS ||--|{ FLIGHT_SEATS : inventory
    FLIGHTS ||--|{ FLIGHT_FARES : offers
    FLIGHTS ||--o{ BOOKINGS : booked
    BOOKINGS ||--|{ BOOKING_PASSENGERS : contains
    FLIGHT_SEATS ||--o{ BOOKING_PASSENGERS : historical
    FLIGHT_FARES ||--o{ BOOKING_PASSENGERS : selected
    BOOKING_PASSENGERS ||--|{ BOOKING_PRICE_LINES : snapshot
    BOOKINGS ||--o{ PAYMENTS : attempts
    PAYMENTS ||--o{ REFUNDS : returns
    BOOKING_PASSENGERS ||--o{ REFUNDS : allocation
    BOOKING_PASSENGERS ||--o| TICKETS : issued
    BOOKING_PASSENGERS ||--o{ CHECK_INS : history
    USERS ||--o{ NOTIFICATIONS : receives
    USERS ||--o{ OPERATION_REQUESTS : retries
    SIMULATED_CHARGES ||--o{ SIMULATED_REFUNDS : returns
```

Симулятор связан с Payment/Refund через передаваемые глобальные UUID операции, а не каскадным FK к бизнес-таблицам. Он должен сохранять успех независимо от последующего сбоя финализации заказа.

## 4. Создание заказа

```mermaid
sequenceDiagram
    participant C as Клиент
    participant S as BookingService
    participant DB as PostgreSQL
    C->>S: flight, passengers, seats, clientRequestId
    S->>DB: BEGIN; lock User, Flight
    S->>DB: Повтор ключа? Сроки? Лимит? Цена? Все места?
    alt Все проверки пройдены
        S->>DB: Booking + passengers + price lines + HELD + notification
        S->>DB: COMMIT
        S-->>C: PENDING и holdUntil
    else Ошибка
        S->>DB: ROLLBACK
        S-->>C: Ошибка без частичного удержания
    end
```

## 5. Оплата и поздний результат

```mermaid
sequenceDiagram
    participant C as Клиент
    participant S as PaymentService
    participant DB as PostgreSQL
    participant G as PaymentGateway
    C->>S: Оплатить + Idempotency-Key
    S->>DB: TX A: lock Flight, Booking; проверить повтор, сроки
    S->>DB: PROCESSING + protectedUntil; COMMIT
    S->>G: charge(paymentId, amount), вне TX A
    G-->>S: Успех / отказ / неизвестно
    S->>DB: TX B: lock Flight, Booking, Payment; перечитать состояние
    alt Успех и заказ ещё допустим
        S->>DB: SUCCEEDED + CONFIRMED + BOOKED + tickets + notification
    else Успех после отмены/expiry
        S->>DB: SUCCEEDED + REQUESTED refund; заказ не восстанавливать
    else Нет определённого результата
        S->>DB: Оставить PROCESSING; позже lookup
    end
    S->>DB: COMMIT
    S-->>C: Текущее состояние
```

Определённый отказ даёт FAILED, снимает платёжное продление и проверяет исходный holdUntil. Возврат исполняется таким же трёхфазным способом: резерв в БД → вызов вне транзакции → фиксация/lookup результата. Финализация и фоновые задачи используют одинаковые правила, а не отдельную упрощённую логику.

## 6. Состояние заказа

```mermaid
stateDiagram-v2
    [*] --> PENDING
    PENDING --> CONFIRMED: Допустимый успех оплаты
    PENDING --> EXPIRED: Эффективный срок прошёл
    PENDING --> CANCELLED: Отмена / перенос
    CONFIRMED --> CONFIRMED: Отмена части пассажиров
    CONFIRMED --> CANCELLED: Отменены все пассажиры / рейс
    EXPIRED --> [*]
    CANCELLED --> [*]
```

Поздний успех оплаты не является переходом EXPIRED/CANCELLED → CONFIRMED. Это изменение Payment и создание Refund. Частичная отмена меняет пассажиров, даже когда статус Booking остаётся CONFIRMED.
