@startuml
!include https://raw.githubusercontent.com/plantuml-stdlib/C4-PlantUML/master/C4_Context.puml

LAYOUT_WITH_LEGEND()

title Здоровье+ — C4 Level 1: снижение нагрузки на 1С «Полисы»

Person(patient, "Пациент",
    "Пользователь мобильного приложения")

System_Boundary(healthplus, "Здоровье+ — мобильный контур") {

    System(app, "Мобильное приложение",
        "iOS / Android")

    System(api, "API Gateway",
        "HTTPS/REST, JWT, Rate Limiting, Request ID")

    System(policy, "Policy Service",
        "Проверка полиса, Retry, Circuit Breaker, Fallback")

    System(cache, "Policy Cache",
        "Redis, кэш результатов проверки полиса")
}

System_Ext(onec, "1С: Полисы",
    "Legacy-система. Проверка валидности полиса ДМС")

System_Ext(schedule, "Расписание",
    "Java / REST / PostgreSQL")

System_Ext(crm, "CRM",
    "Java / REST / PostgreSQL")

Rel(patient, app,
    "Использует")

Rel(app, api,
    "HTTPS/REST")

Rel(api, policy,
    "REST: проверка полиса")

Rel(policy, cache,
    "GET/SET: результат проверки")

Rel(policy, onec,
    "HTTP: проверка полиса\nтолько при Cache MISS")

Rel(api, schedule,
    "REST: запись к врачу")

Rel(api, crm,
    "REST: данные пациента")

note right of policy
Защита 1С:

Timeout = 3 sec
Retry = 2
Backoff = 1s / 3s
Circuit Breaker
Rate Limit
Concurrency Limit
Fallback
end note

note right of cache
Основной механизм разгрузки:

Cache HIT
→ 1С не вызывается

Cache MISS
→ обращение к 1С

TTL определяется
бизнес-требованиями
end note

@enduml
