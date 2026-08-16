# API для мобильного приложения — OpenAPI
1. Назначение API
Для мобильного приложения предоставляется безопасный API для получения пациентом списка его записей к врачам.

Целевой контур:
```text
Mobile App
    ↓ HTTPS
API Gateway
    ↓
Mobile Backend / BFF
    ↓
Integration Layer
    ↓
Расписание / CRM
```
Мобильное приложение не обращается напрямую к Legacy-системам. API Gateway выполняет аутентификацию, rate limiting и защитные проверки, а BFF формирует мобильный контракт.

API версионируется в URL:

/api/v1/appointments
2. OpenAPI 3.2
openapi: 3.2.0

info:
  title: Health+ Mobile API
  description: API мобильного приложения сети клиник «Здоровье+».
  version: 1.0.0

servers:
  - url: https://api.healthplus.ru

tags:
  - name: Appointments
    description: Записи пациента к врачам

paths:
  /api/v1/appointments:
    get:
      tags:
        - Appointments
      summary: Получить список записей пациента
      description: |
        Возвращает список записей текущего аутентифицированного пациента.
        Идентификатор пациента определяется из access token и не передаётся
        клиентом в query-параметрах.
      operationId: getPatientAppointments

      security:
        - bearerAuth: []

      parameters:
        - name: status
          in: query
          description: Фильтр по статусу записи.
          required: false
          schema:
            type: string
            enum:
              - upcoming
              - confirmed
              - cancelled
              - completed
            default: upcoming

        - name: limit
          in: query
          description: Максимальное количество записей в ответе.
          required: false
          schema:
            type: integer
            format: int32
            minimum: 1
            maximum: 100
            default: 20

        - name: offset
          in: query
          description: Количество пропускаемых записей.
          required: false
          schema:
            type: integer
            format: int32
            minimum: 0
            default: 0

      responses:
        '200':
          description: Список записей пациента успешно получен.
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/AppointmentsResponse'
              example:
                data:
                  - id: APT-20260820-001245
                    status: upcoming
                    appointmentDateTime: '2026-08-20T14:30:00+03:00'
                    durationMinutes: 30
                    doctor:
                      id: DOC-10234
                      firstName: Иван
                      lastName: Петров
                      middleName: Сергеевич
                      specialty: Терапевт
                    clinic:
                      id: CLN-MSK-001
                      name: Здоровье+ Москва
                      city: Москва
                      address: 'ул. Примерная, д. 10'
                    service:
                      id: SRV-001
                      name: Первичная консультация терапевта
                    paymentStatus: paid
                    labReservation:
                      id: LAB-20260820-00021
                      status: reserved
                      dateTime: '2026-08-20T15:30:00+03:00'
                pagination:
                  limit: 20
                  offset: 0
                  total: 1
                  hasNext: false

        '401':
          description: Отсутствует или недействителен access token.
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
              example:
                error:
                  code: UNAUTHORIZED
                  message: Authentication is required
                  requestId: req-01J7XYZ123

        '500':
          description: Внутренняя ошибка сервера.
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
              example:
                error:
                  code: INTERNAL_SERVER_ERROR
                  message: An unexpected error occurred
                  requestId: req-01J7XYZ123

components:
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
      description: |
        JWT access token пациента. Идентификатор пациента
        извлекается из claims токена.

  schemas:
    AppointmentsResponse:
      type: object
      required:
        - data
        - pagination
      properties:
        data:
          type: array
          items:
            $ref: '#/components/schemas/Appointment'
        pagination:
          $ref: '#/components/schemas/Pagination'

    Appointment:
      type: object
      required:
        - id
        - status
        - appointmentDateTime
        - doctor
        - clinic
        - service
        - paymentStatus
      properties:
        id:
          type: string
          example: APT-20260820-001245
        status:
          type: string
          enum:
            - upcoming
            - confirmed
            - cancelled
            - completed
          example: upcoming
        appointmentDateTime:
          type: string
          format: date-time
          description: Дата и время приёма в часовом поясе клиники.
          example: '2026-08-20T14:30:00+03:00'
        durationMinutes:
          type: integer
          minimum: 1
          example: 30
        doctor:
          $ref: '#/components/schemas/Doctor'
        clinic:
          $ref: '#/components/schemas/Clinic'
        service:
          $ref: '#/components/schemas/Service'
        paymentStatus:
          type: string
          enum:
            - pending
            - paid
            - failed
            - refunded
          example: paid
        labReservation:
          nullable: true
          allOf:
            - $ref: '#/components/schemas/LabReservation'

    Doctor:
      type: object
      required:
        - id
        - firstName
        - lastName
        - specialty
      properties:
        id:
          type: string
          example: DOC-10234
        firstName:
          type: string
          example: Иван
        lastName:
          type: string
          example: Петров
        middleName:
          type: string
          nullable: true
          example: Сергеевич
        specialty:
          type: string
          example: Терапевт

    Clinic:
      type: object
      required:
        - id
        - name
        - city
        - address
      properties:
        id:
          type: string
          example: CLN-MSK-001
        name:
          type: string
          example: Здоровье+ Москва
        city:
          type: string
          example: Москва
        address:
          type: string
          example: 'ул. Примерная, д. 10'

    Service:
      type: object
      required:
        - id
        - name
      properties:
        id:
          type: string
          example: SRV-001
        name:
          type: string
          example: Первичная консультация терапевта

    LabReservation:
      type: object
      required:
        - id
        - status
        - dateTime
      properties:
        id:
          type: string
          example: LAB-20260820-00021
        status:
          type: string
          enum:
            - reserved
            - cancelled
            - completed
          example: reserved
        dateTime:
          type: string
          format: date-time
          example: '2026-08-20T15:30:00+03:00'

    Pagination:
      type: object
      required:
        - limit
        - offset
        - total
        - hasNext
      properties:
        limit:
          type: integer
          example: 20
        offset:
          type: integer
          example: 0
        total:
          type: integer
          example: 125
        hasNext:
          type: boolean
          example: true

    ErrorResponse:
      type: object
      required:
        - error
      properties:
        error:
          type: object
          required:
            - code
            - message
            - requestId
          properties:
            code:
              type: string
              example: UNAUTHORIZED
            message:
              type: string
              example: Authentication is required
            requestId:
              type: string
              description: Идентификатор запроса для логов и distributed tracing.
              example: req-01J7XYZ123
3. Пагинация
Для MVP используется offset-based pagination:

GET /api/v1/appointments?status=upcoming&limit=20&offset=0

GET /api/v1/appointments?status=upcoming&limit=20&offset=20

GET /api/v1/appointments?status=upcoming&limit=20&offset=40
Параметры:

limit — количество записей на странице;
offset — количество пропускаемых записей;
total — общее количество записей;
hasNext — наличие следующей страницы.
Для данного endpoint offset-пагинация подходит, поскольку список записей одного пациента обычно относительно небольшой. Ограничение limit <= 100 защищает backend от чрезмерно больших запросов.

Для стабильности страниц используется детерминированная сортировка:

appointmentDateTime ASC, id ASC
Если в будущем объём данных существенно вырастет, можно перейти на cursor-based pagination.

4. Версионирование API
Используется версия в URL:

/api/v1/appointments
Следующая несовместимая версия:

/api/v2/appointments
Такой подход выбран потому, что версия явно видна разработчикам и операторам, позволяет поддерживать несколько версий одновременно и упрощает маршрутизацию через API Gateway.

Обратно совместимые изменения, например добавление необязательного поля, не требуют новой major-версии. Изменения, нарушающие контракт, должны выпускаться в новой версии.

5. Аутентификация и безопасность
API требует:

Authorization: Bearer <JWT>
patientId не передаётся клиентом:

/api/v1/appointments?patientId=123
не используется.

Пациент определяется из JWT claims. Это снижает риск получения данных другого пациента через подмену идентификатора.

С учётом SD-007 («Врач видит чужого пациента») авторизация должна проверяться на серверной стороне, а не только в мобильном приложении.

API Gateway дополнительно должен обеспечивать:

TLS;
rate limiting;
проверку JWT;
request size limits;
request/correlation ID;
аудит обращений к персональным данным.
6. HTTP-коды
200 OK
{
  "data": [],
  "pagination": {
    "limit": 20,
    "offset": 0,
    "total": 0,
    "hasNext": false
  }
}
401 Unauthorized
{
  "error": {
    "code": "UNAUTHORIZED",
    "message": "Authentication is required",
    "requestId": "req-01J7XYZ123"
  }
}
500 Internal Server Error
{
  "error": {
    "code": "INTERNAL_SERVER_ERROR",
    "message": "An unexpected error occurred",
    "requestId": "req-01J7XYZ123"
  }
}
requestId позволяет связать ошибку мобильного клиента с логами и distributed tracing, что напрямую помогает решить проблему INT-007.

7. Итоговое архитектурное решение
                    Mobile App
                        |
                     HTTPS
                        |
                        v
                 +--------------+
                 | API Gateway  |
                 |              |
                 | JWT          |
                 | Rate Limit   |
                 | Request ID   |
                 +------+-------+
                        |
                        | REST
                        v
                 +--------------+
                 | Mobile BFF   |
                 +------+-------+
                        |
                        v
                 +--------------+
                 | Integration  |
                 | Layer        |
                 +------+-------+
                        |
                        v
                 +--------------+
                 | Расписание   |
                 +--------------+
API /api/v1/appointments предоставляет стабильный контракт мобильному приложению и скрывает внутреннюю реализацию ИТ-ландшафта.

Мобильное приложение не должно знать, где физически находятся данные и какие Legacy-системы используются для их получения. BFF отвечает за агрегацию и адаптацию данных, а API Gateway — за cross-cutting concerns: аутентификацию, rate limiting, трассировку и защиту внешнего API.

Ключевые решения
Область	Решение
Endpoint	GET /api/v1/appointments
Фильтрация	status=upcoming
Пагинация	limit + offset
Максимальный размер страницы	100
Сортировка	appointmentDateTime ASC, id ASC
Аутентификация	Bearer JWT
Идентификация пациента	Из JWT, не из запроса
Версионирование	URL /api/v1
Backend	Mobile BFF
Внешний вход	API Gateway
Rate limiting	API Gateway
Tracing	requestId / correlation ID
Ошибки	Единый ErrorResponse
Legacy isolation	Integration Layer
