# Архитектурное решение

Целевая архитектура строится вокруг нового **мобильного канала**, который не должен напрямую зависеть от Legacy-систем.

Основной принцип:

> **Mobile App → API Gateway → Mobile Backend / BFF → Integration Layer → существующие ИТ-системы**

Это позволяет изолировать мобильное приложение от нестабильности 1С, LIS, старого платёжного шлюза и других систем, а также централизовать безопасность, rate limiting, идемпотентность, timeout, retry, observability и аудит.

### Новые системы

| Новая система | Назначение |
|---|---|
| **API Gateway** | Единая точка входа мобильного приложения; TLS termination, authentication, rate limiting, защита API, correlation ID, маршрутизация |
| **Mobile Backend / BFF** | API, адаптированное под мобильные сценарии; агрегирует данные нескольких систем и скрывает внутренний ИТ-ландшафт |
| **Integration Layer** | Изоляция мобильного канала от Legacy; адаптеры и orchestration для расписания, 1С, LIS, CRM и платежей |
| **Event Bus** | Асинхронная передача событий между системами; decoupling и буферизация пиковых нагрузок |
| **Policy Service / DMS Read Model** | Быстрое чтение актуального состояния полиса без обращения к 1С на каждый запрос |
| **Notification Service** | Управление уведомлениями мобильного приложения и интеграция с существующим RabbitMQ/провайдерами |
| **Identity & Access Management** | Аутентификация пользователей мобильного приложения, управление токенами и авторизацией |
| **Observability Platform** | Centralized logging, metrics и distributed tracing |

---

##  Интеграционные паттерны и архитектурные стили

### 1. Mobile App → API Gateway

**Паттерн:** API Gateway.

**Стиль:** синхронный request/response API.

**Протокол:** HTTPS + JSON, OAuth 2.0/OIDC.

API Gateway является единой точкой входа в мобильный backend.

Через него реализуются:

- authentication;
- authorization;
- rate limiting;
- защита от DDoS/аномального трафика;
- correlation ID;
- маршрутизация;
- ограничение размера запросов;
- базовая валидация;
- аудит.

**Почему:**

Мобильное приложение не должно знать адреса и особенности внутренних систем. Кроме того, Gateway позволяет ограничивать нагрузку на Legacy. Это особенно важно для 1С, которая уже имеет проблемы при нагрузке более 50 RPS.

---

### 2. API Gateway → Mobile Backend / BFF

**Паттерн:** Backend for Frontend.

**Стиль:** синхронный REST API.

BFF предоставляет мобильному приложению API, соответствующее его бизнес-сценариям.

Например:

```text
GET /mobile/doctors/{id}/slots
POST /mobile/bookings
POST /mobile/payments
GET /mobile/analyses
GET /mobile/policies
```

BFF скрывает внутренние системы и позволяет мобильному приложению получать агрегированный ответ вместо нескольких последовательных вызовов.

**Почему:**

Если мобильное приложение напрямую вызывает расписание, CRM, 1С и платежи, каждый Legacy-контракт становится частью мобильного API. BFF снижает связанность и позволяет менять внутреннюю архитектуру без выпуска новой версии приложения.

---

### 3. BFF → Integration Layer

**Паттерны:**

- API orchestration;
- Anti-Corruption Layer;
- Adapter;
- Circuit Breaker;
- Retry;
- Timeout;
- Bulkhead.

**Стиль:** преимущественно синхронный REST/gRPC для пользовательских операций.

Integration Layer преобразует внутреннюю модель мобильного приложения в модели существующих систем.

Например:

```text
Mobile:
POST /bookings

        ↓

Integration Layer

        ↓

Schedule:
POST /appointments
```

При этом мобильный API не зависит от конкретной структуры REST API расписания.

**Почему:**

Legacy-системы имеют разные технологии и контракты: Java/REST, 1С/HTTP, SOAP, FTP. Integration Layer изолирует эти различия.

---

## 4. Запись к врачу

### Целевой поток

```text
Мобильное приложение
        ↓
API Gateway
        ↓
Mobile BFF
        ↓
Integration Layer
        ↓
Расписание
```

**Стиль:** synchronous request/response.

**Паттерны:**

- Idempotent Consumer;
- Idempotency Key;
- Timeout;
- Retry только для безопасных операций;
- Circuit Breaker.

Для каждой команды записи приложение передаёт уникальный:

```text
Idempotency-Key: <UUID>
```

Integration Layer сохраняет результат обработки ключа.

Если пользователь повторит запрос из-за timeout, операция не создаст вторую запись.

### Почему не делать запись асинхронной полностью

Запись должна дать пользователю понятный результат:

```text
Записан
```

или

```text
Не записан
```

Поэтому основной command-flow остаётся синхронным.

Асинхронно можно отправлять уже **события после успешной записи**:

```text
BookingCreated
BookingCancelled
BookingRescheduled
```

---

## 5. Оплата

### Целевой поток

```text
Mobile App
    ↓
API Gateway
    ↓
BFF
    ↓
Payment Orchestrator
    ↓
Платёжный шлюз
    ↓
Webhook
    ↓
Payment Orchestrator
    ↓
PaymentConfirmed
    ↓
Event Bus
```

**Стиль:** комбинация synchronous + asynchronous.

Синхронно:

- создаётся платёж;
- пользователю возвращается `paymentId`.

Асинхронно:

- платёжный провайдер присылает callback/webhook;
- система получает окончательный статус;
- публикуется событие `PaymentConfirmed` или `PaymentFailed`.

### Почему комбинация

Синхронный REST нельзя считать надёжным источником окончательного состояния платежа: текущая система уже имеет проблему `INT-001`, когда деньги списаны, а статус не зафиксирован.

Поэтому вводится **state machine платежа**:

```text
NEW
 ↓
PENDING
 ↓
CONFIRMED
```

или:

```text
PENDING → FAILED
```

При timeout система не делает слепой повторный платёж, а сначала проверяет состояние существующего `paymentId`.

Это предотвращает двойное списание.

---

## 6. Проверка полиса ДМС

Текущая 1С является одним из главных bottleneck.

Поэтому мобильное приложение **не должно обращаться к 1С при каждом чтении полиса**.

### Новый поток

```text
1С: Полисы
      ↓
Policy Adapter
      ↓
Policy Service / DMS Read Model
      ↑
      |
Mobile BFF
      ↑
      |
Mobile App
```

**Стиль:** CQRS / Read Model + asynchronous synchronization.

Для чтения мобильное приложение получает данные из оптимизированного Read Model.

1С используется как источник актуального состояния полиса, а не как высоконагруженный API для мобильных пользователей.

### Почему

Это защищает 1С от нагрузки мобильного приложения и устраняет проблему `INT-003` с пределом около 50 RPS.

Для критической операции, например непосредственной проверки возможности оказания услуги, допускается controlled synchronous fallback в 1С с:

- timeout;
- circuit breaker;
- rate limiting;
- кэшированием;
- ограничением количества одновременных запросов.

---

## 7. Результаты анализов

Текущая цепочка:

```text
LIS → FTP → CRM
```

имеет проблемы с:

- FTP;
- повторной передачей;
- дублированием XML;
- задержками;
- ручным восстановлением.

Целевая схема:

```text
LIS
 ↓
LIS Adapter
 ↓
Event Bus
 ↓
Results Processing
 ↓
CRM / Results Read Model
 ↓
Mobile BFF
 ↓
Mobile App
```

**Стиль:** Event-Driven Architecture.

**Паттерны:**

- Adapter;
- Idempotent Consumer;
- Retry;
- Dead Letter Queue;
- Event-driven integration;
- eventual consistency.

Каждому результату должен соответствовать уникальный идентификатор:

```text
resultId
```

Повторная доставка одного и того же результата не должна создавать дубль.

### Почему асинхронно

Результат анализа не требует мгновенного синхронного ответа от LIS в момент пользовательского запроса. Это естественный asynchronous business process.

Кроме того, Event Bus позволяет переживать временную недоступность CRM и сглаживать пики нагрузки.

---

## 8. Уведомления

Целевая схема:

```text
BookingCreated
PaymentConfirmed
AnalysisReady
        ↓
     Kafka
        ↓
Notification Service
        ↓
RabbitMQ / provider APIs
        ↓
Push / SMS / E-mail
```

**Стиль:** Event-Driven Architecture.

**Почему не отправлять уведомление синхронно из BFF**

Отправка push/SMS/email не должна увеличивать latency пользовательского запроса.

Например:

```text
POST /booking
       ↓
Booking confirmed
       ↓
HTTP 200
```

а уже после этого:

```text
BookingCreated
       ↓
Notification Service
       ↓
Push
```

Таким образом, недоступность SMS-провайдера не ломает запись пациента.

---

## 9. Почему Kafka + RabbitMQ

В целевой архитектуре предлагается **не удалять RabbitMQ сразу**, а использовать его как существующий транспорт уведомлений.

Kafka используется как корпоративный Event Bus для новых интеграционных событий:

```text
Kafka
 ├── BookingCreated
 ├── BookingCancelled
 ├── PaymentConfirmed
 ├── PaymentFailed
 ├── AnalysisReady
 └── PolicyUpdated
```

RabbitMQ сохраняется на первом этапе:

```text
Notification Service
        ↓
RabbitMQ
        ↓
SMS / Email / Push
```

Это снижает риск миграции в рамках MVP.

В дальнейшем RabbitMQ можно заменить или вывести из критического пути после стабилизации нового notification-контура.

---

## 10. Observability

Для всех новых компонентов используется единый механизм наблюдаемости:

- distributed tracing;
- metrics;
- centralized logging;
- correlation ID;
- health checks;
- alerting.

Например, один запрос получает:

```text
traceId = 7f8a...
```

и этот идентификатор проходит через:

```text
Mobile
 → API Gateway
 → BFF
 → Integration Layer
 → Schedule
```

Это непосредственно устраняет проблему `INT-007`.

---

## 11. Основные архитектурные стили

| Стиль | Где используется | Почему |
|---|---|---|
| **API Gateway** | Mobile → Gateway | Единая защищённая точка входа и управление нагрузкой |
| **Backend for Frontend** | Gateway → BFF | API, адаптированный под мобильные сценарии |
| **Synchronous REST/gRPC** | Запись, получение слотов, инициирование платежа | Пользователю нужен быстрый результат операции |
| **Event-Driven Architecture** | Уведомления, результаты анализов, события платежей | Decoupling, буферизация и отказоустойчивость |
| **CQRS / Read Model** | Полисы, результаты анализов | Быстрое чтение без нагрузки на Legacy |
| **Anti-Corruption Layer** | Integration Layer → Legacy | Изоляция мобильного API от Legacy-моделей |
| **Adapter Pattern** | LIS, 1С, RabbitMQ, платёжный шлюз | Разные протоколы и технологии |
| **Circuit Breaker** | 1С, платежи, расписание, CRM | Защита от каскадных отказов |
| **Retry + Dead Letter Queue** | Асинхронные интеграции | Автоматическое восстановление после временных ошибок |
| **Idempotency** | Запись и платежи | Защита от повторной обработки |
| **Bulkhead** | Критичные бизнес-потоки | Изоляция отказов |
| **Rate Limiting** | API Gateway и Legacy adapters | Защита downstream-систем |
| **Observability** | Весь новый контур | Снижение MTTR и контроль SLA |

---

## 12. Соответствие бизнес-требованиям

| Требование | Архитектурное решение |
|---|---|
| MVP через 3 месяца | API Gateway + BFF + Integration Layer позволяют не менять мобильный клиент при проблемах Legacy |
| 500 одновременных пользователей | Horizontal scaling Gateway/BFF + rate limiting + connection pooling |
| Снижение нагрузки на колл-центр на 30% | Надёжный self-service booking |
| Онлайн-запись 70% визитов | Масштабируемый booking API + оптимизация расписания |
| До 50 000 пользователей | Горизонтальное масштабирование Gateway/BFF/Integration Layer/Event Bus |
| p99 < 1 секунды | Кэширование, Read Models, асинхронные операции и отсутствие Legacy в некритичных путях |
| Доступность 99,95% | Circuit Breaker, retry, redundancy, graceful degradation |
| Защита 1С | Policy Read Model + rate limiting + controlled fallback |
| Надёжные платежи | Payment Orchestrator + idempotency + webhook + state machine |
| Надёжные уведомления | Event Bus + Notification Service + RabbitMQ + retry |
| Актуальные анализы | Event-driven LIS integration + deduplication + Read Model |
| Безопасность | API Gateway + OAuth2/OIDC + централизованная авторизация |
| Быстрое расследование сбоев | Distributed tracing + metrics + centralized logging |

