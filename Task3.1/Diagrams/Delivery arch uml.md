@startuml
title Здоровье+ — доставка результатов анализов в мобильное приложение

skinparam componentStyle rectangle
skinparam shadowing false

rectangle "Лабораторное\nоборудование" as Equipment
rectangle "LIS\nLaboratory Information System" as LIS
rectangle "LIS Integration Adapter\nAnti-Corruption Layer" as Adapter
queue "Kafka\nlab-results" as Kafka

rectangle "Result Processing\nService" as ResultService
database "Results Read Model\n(CQRS)" as ReadModel

rectangle "CRM Integration\nConsumer" as CRMConsumer
rectangle "CRM" as CRM

rectangle "Notification\nService" as Notification
queue "Notification Retry / DLQ" as DLQ
rectangle "Push / SMS / E-mail\nProviders" as Providers

rectangle "Mobile Backend / BFF" as BFF
rectangle "API Gateway" as Gateway
rectangle "Mobile App" as Mobile

Equipment --> LIS : Результат исследования\nFTP
LIS --> Adapter : Результат\nREST / SOAP

Adapter --> Kafka : LabResultReady\nEvent

Kafka --> ResultService : Consume\nConsumer Group
Kafka --> CRMConsumer : Consume\nConsumer Group

ResultService --> ReadModel : Upsert result\nIdempotent Consumer

CRMConsumer --> CRM : Update result\nREST

ReadModel --> BFF : Read results\nREST
BFF --> Gateway : API
Gateway --> Mobile : HTTPS

ResultService --> Kafka : AnalysisReadyForPatient\nEvent

Kafka --> Notification : Consume event

Notification --> Providers : Push / SMS / E-mail

Notification --> DLQ : Failed delivery\nRetry exhausted

note right of Adapter
Паттерны:
- Adapter
- Anti-Corruption Layer
- Retry
- Idempotency
- Correlation ID
end note

note right of Kafka
Event-Driven Architecture

Преимущества:
- decoupling
- buffering
- backpressure
- replay
- horizontal scaling
- durable events
end note

note right of ResultService
Idempotent Consumer

resultId используется
как уникальный ключ.

Повторное событие
не создаёт дубль.
end note

note right of ReadModel
CQRS / Read Model

Оптимизировано
для быстрого чтения
мобильным приложением.

Не нагружает CRM.
end note

note bottom of Notification
Push — дополнительный канал информирования.

Доступность результата в API
не зависит от успешности Push.
end note

@enduml
