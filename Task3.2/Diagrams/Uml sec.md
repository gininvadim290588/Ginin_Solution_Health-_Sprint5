@startuml
title Здоровье+ — Запись к врачу с оплатой и резервированием лаборатории

actor "Пациент" as Patient
participant "Mobile App" as App
participant "API Gateway" as Gateway
participant "Booking Saga\nOrchestrator" as Saga
participant "Policy Service /\n1С: Полисы Adapter" as Policy
participant "Расписание" as Schedule
participant "LIS Adapter" as LISAdapter
participant "LIS" as LIS
participant "Payment Adapter" as PayAdapter
participant "Платёжный шлюз" as Payment
participant "Notification Service" as Notify

Patient -> App : Выбор врача, слота\nи анализа
App -> Gateway : POST /booking\nIdempotency-Key
Gateway -> Saga : Start Booking Saga

note right of Gateway
Idempotency-Key:
уникальный ключ операции
end note

Saga -> Policy : Проверить полис
Policy -> Policy : Проверка валидности
Policy --> Saga : Policy valid

Saga -> Schedule : Reserve doctor slot\nbookingId
Schedule -> Schedule : Проверка доступности\n+ идемпотентная запись
Schedule --> Saga : Slot reserved

Saga -> LISAdapter : Reserve lab slot\nreservationId
LISAdapter -> LIS : Reserve lab slot
LIS --> LISAdapter : Lab slot reserved
LISAdapter --> Saga : Lab slot reserved

Saga -> PayAdapter : Create / authorize payment\npaymentId
PayAdapter -> Payment : Payment request\nidempotency key
Payment --> PayAdapter : Payment authorized
PayAdapter --> Saga : Payment successful

Saga -> Schedule : Confirm booking\nbookingId
Schedule --> Saga : Booking confirmed

Saga -> LISAdapter : Confirm lab reservation\nreservationId
LISAdapter -> LIS : Confirm reservation
LIS --> LISAdapter : Reservation confirmed
LISAdapter --> Saga : Lab reservation confirmed

Saga -> Notify : Publish BookingConfirmed
Notify --> App : Push confirmation
App --> Patient : Запись подтверждена\nОплата успешна\nЛаборатория зарезервирована

Saga -> Saga : Saga COMPLETED

@enduml
