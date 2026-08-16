# Выбор поставка результатов анализов в мобильное приложение
Выбранное решение
Предлагается использовать Kafka + Event-Driven Architecture + Idempotent Consumer + CQRS Read Model.

Текущий поток:
Лабораторное оборудование

        ↓ FTP
        
       LIS
       
        ↓ REST
        
       CRM
       
        ↓
Мобильное приложение

Целевой поток:

Лабораторное оборудование
        ↓ FTP
       LIS
        ↓ REST/SOAP
LIS Integration Adapter
        ↓ LabResultReady
      Kafka
        ├───────────────┐
        ↓               ↓
Result Processing   CRM Consumer
        ↓               ↓
Results Read Model   CRM
        ↓
   Mobile BFF
        ↓
   Mobile App
После успешной записи результата в Read Model публикуется AnalysisReadyForPatient, которое обрабатывается Notification Service и приводит к отправке push.

Сравнение вариантов
Критерий	Синхронный REST	Kafka / Event-Driven	Polling
Гарантия доставки	Средняя, требует retry	Высокая при корректной обработке offset	Средняя
Защита от дублей	Отдельная реализация	Idempotent Consumer + resultId	Отдельная реализация
Пиковая нагрузка	Ограничена цепочкой вызовов	Буферизация и backpressure	Постоянная дополнительная нагрузка
Retry	Вручную	Retry + DLQ	Повторный polling
Масштабирование	Ограниченное	Горизонтальное	Среднее
Replay	Нет	Да	Ограниченно
Latency	Низкая	Низкая	Зависит от интервала
Сложность эксплуатации	Низкая	Средняя	Низкая/средняя
Соответствие требованиям	Частичное	Наилучшее	Низкое
Основные паттерны
Adapter + Anti-Corruption Layer
LIS Integration Adapter изолирует мобильный и современный контур от SOAP/FTP/REST Legacy-интерфейсов LIS.

Event-Driven Architecture
Событие LabResultReady публикуется в Kafka. LIS не зависит от того, сколько потребителей существует: CRM, Read Model, Notification Service и другие сервисы могут независимо подписываться на событие.

Idempotent Consumer
Каждый результат имеет уникальный resultId.

IF resultId already exists
    → ignore duplicate
ELSE
    → save result
Это устраняет проблему INT-008 с дублированием результатов.

Retry + DLQ
Kafka → Consumer
          ↓
       success ──→ processed

       error
          ↓
        retry
          ↓
      retry exhausted
          ↓
         DLQ
Временные ошибки обрабатываются автоматически с exponential backoff. Неисправимые сообщения попадают в DLQ для расследования и повторной обработки.

CQRS / Read Model
Результаты для мобильного приложения хранятся в отдельной Results Read Model.

Это позволяет:

быстро читать результаты;
не создавать нагрузку на CRM;
горизонтально масштабировать Mobile API;
не зависеть от доступности CRM при чтении.
Контракт события
{
  "eventId": "01J...",
  "eventType": "LabResultReady",
  "eventVersion": "1.0",
  "occurredAt": "2026-08-16T10:15:23Z",
  "patientId": "123456",
  "orderId": "LAB-987654",
  "resultId": "RES-123456",
  "source": "LIS",
  "correlationId": "COR-123456"
}
Поток данных
Лабораторное оборудование завершает исследование.
LIS получает результат.
LIS Integration Adapter принимает результат.
Формируется событие LabResultReady.
Событие публикуется в Kafka.
Kafka сохраняет событие.
Result Processing Service получает событие.
Проверяется уникальность resultId.
Результат записывается в Results Read Model.
Публикуется AnalysisReadyForPatient.
Notification Service отправляет push.
Пациент открывает приложение.
Mobile BFF получает результат из Read Model.
Результат отображается пациенту.
PlantUML — блок-схема
@startuml
title Здоровье+ — доставка результатов анализов в мобильное приложение

skinparam componentStyle rectangle
skinparam shadowing false

rectangle "Лабораторное оборудование" as Equipment
rectangle "LIS" as LIS
rectangle "LIS Integration Adapter
Anti-Corruption Layer" as Adapter
queue "Kafka
lab-results" as Kafka

rectangle "Result Processing
Service" as ResultService
database "Results Read Model
(CQRS)" as ReadModel

rectangle "CRM Integration
Consumer" as CRMConsumer
rectangle "CRM" as CRM

rectangle "Notification
Service" as Notification
queue "Retry / DLQ" as DLQ
rectangle "Push / SMS / E-mail
Providers" as Providers

rectangle "Mobile Backend / BFF" as BFF
rectangle "API Gateway" as Gateway
rectangle "Mobile App" as Mobile

Equipment --> LIS : Результат исследования
FTP
LIS --> Adapter : Результат
REST / SOAP
Adapter --> Kafka : LabResultReady
Event

Kafka --> ResultService : Consume
Consumer Group
Kafka --> CRMConsumer : Consume
Consumer Group

ResultService --> ReadModel : Upsert
Idempotent Consumer
CRMConsumer --> CRM : Update result
REST

ReadModel --> BFF : Read results
REST
BFF --> Gateway : API
Gateway --> Mobile : HTTPS

ResultService --> Kafka : AnalysisReadyForPatient
Kafka --> Notification : Consume event

Notification --> Providers : Push / SMS / E-mail
Notification --> DLQ : Retry exhausted

note right of Adapter
Adapter / ACL
- изоляция Legacy
- нормализация формата
- correlationId
end note

note right of Kafka
Event-Driven Architecture
- decoupling
- buffering
- backpressure
- replay
- horizontal scaling
end note

note right of ResultService
Idempotent Consumer
resultId = уникальный ключ

Повторное событие
не создаёт дубль.
end note

note right of ReadModel
CQRS / Read Model

Быстрое чтение
без нагрузки на CRM.
end note

note bottom of Notification
Push — только канал информирования.
Результат всегда доступен
через Mobile API.
end note

@enduml
Гарантии решения
Требование	Решение
Не терять результаты при недоступности CRM	Kafka + durable events
Не создавать дубли	Idempotent Consumer + resultId
Переживать пики	Kafka buffering + Consumer Groups
Автоматически восстанавливаться	Retry + exponential backoff
Обрабатывать неисправимые ошибки	DLQ
Быстро отдавать результаты	Results Read Model
Не зависеть от CRM при чтении	CQRS
Уведомлять пациента	Event → Notification Service → Push
Не зависеть от Push	Результат доступен через API
Расследовать проблемы	eventId + correlationId + tracing
Не переписывать Legacy	Adapter / ACL
Архитектурный вывод
Выбранная архитектура — асинхронная Event-Driven архитектура на Kafka с Adapter/Anti-Corruption Layer, Idempotent Consumer, Retry/DLQ и отдельным CQRS Read Model.

Она устраняет основные проблемы текущего LIS → CRM потока: потерю данных, дублирование, отсутствие автоматического восстановления и большие задержки, при этом позволяет независимо масштабировать обработку результатов и мобильный API.
