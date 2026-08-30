### Cпецификация (YAML)

```yaml
openapi: 3.2.0
info:
  title: Здоровье+ – API мобильного приложения
  version: 1.0.0
  description: |
    API‑gateway обслуживает мобильное приложение «Здоровье+».
    Энд‑поинт возвращает список предстоящих записей пациента.
servers:
  - url: https://api.healthplus.ru
    description: Production server
  - url: https://api.staging.healthplus.ru
    description: Staging server

paths:
  /api/v1/appointments:
    get:
      summary: Список предстоящих записей пациента
      description: |
        Возвращает записи только для аутентифицированного пациента.
        Поддерживается фильтрация по статусу и постраничный вывод.
      operationId: getPatientAppointments
      tags:
        - Appointments
      security:
        - bearerAuth: []                     # JWT‑токен
      parameters:
        - name: status
          in: query
          description: |
            Фильтр по статусу записи.  
            `upcoming` – запись в будущем (по‑умолчанию).  
            `completed` – уже прошедшие.  
            `canceled` – отменённые.
          required: false
          schema:
            type: string
            enum: [upcoming, completed, canceled]
            default: upcoming
        - name: limit
          in: query
          description: Максимальное количество записей в ответе (постраничность).
          required: false
          schema:
            type: integer
            minimum: 1
            maximum: 100
            default: 20
        - name: offset
          in: query
          description: Смещение от начала списка (используется совместно с `limit`).
          required: false
          schema:
            type: integer
            minimum: 0
            default: 0
      responses:
        '200':
          description: Успешный ответ – список записей.
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/AppointmentListResponse'
              examples:
                success:
                  summary: Пример ответа 200
                  value:
                    total: 42
                    limit: 20
                    offset: 0
                    data:
                      - appointmentId: "a7d9c1e4-3f2b-11ee-be56-0242ac120002"
                        patientId: "p123456"
                        doctorId: "d987"
                        clinicId: "msk-01"
                        startTime: "2026-09-12T10:00:00+03:00"
                        endTime:   "2026-09-12T10:30:00+03:00"
                        status: "upcoming"
                        serviceCode: "CONSULT"
                        paymentStatus: "paid"
                      - appointmentId: "b3f1a6c8-3f2b-11ee-be56-0242ac120002"
                        patientId: "p123456"
                        doctorId: "d654"
                        clinicId: "spb-02"
                        startTime: "2026-09-20T14:00:00+03:00"
                        endTime:   "2026-09-20T14:30:00+03:00"
                        status: "upcoming"
                        serviceCode: "LAB_BIO"
                        paymentStatus: "unpaid"
        '401':
          description: Неавторизованный запрос – отсутствует или просрочен JWT.
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
              examples:
                unauthorized:
                  summary: Пример 401
                  value:
                    code: 401
                    message: "Invalid or missing authentication token."
        '500':
          description: Внутренняя ошибка сервера (например, проблема с базой данных или внешними сервисами).
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
              examples:
                serverError:
                  summary: Пример 500
                  value:
                    code: 500
                    message: "Unexpected error while fetching appointments."

components:
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
      description: |
        JWT‑токен, получаемый после авторизации (логин/пароль или SSO).  
        Токен должен быть передан в заголовке `Authorization: Bearer <token>`.

  schemas:
    Appointment:
      type: object
      required:
        - appointmentId
        - patientId
        - doctorId
        - clinicId
        - startTime
        - endTime
        - status
        - serviceCode
        - paymentStatus
      properties:
        appointmentId:
          type: string
          format: uuid
          description: Уникальный идентификатор записи.
        patientId:
          type: string
          description: Идентификатор пациента в CRM.
        doctorId:
          type: string
          description: Идентификатор врача.
        clinicId:
          type: string
          description: Идентификатор филиала (Москва, СПБ, и т.д.).
        startTime:
          type: string
          format: date-time
          description: Начало приёма (в локальном часовом поясе клиники).
        endTime:
          type: string
          format: date-time
          description: Окончание приёма.
        status:
          type: string
          enum: [upcoming, completed, canceled]
          description: Текущий статус записи.
        serviceCode:
          type: string
          description: Код услуги (CONSULT, LAB_BIO, etc.).
        paymentStatus:
          type: string
          enum: [paid, unpaid, pending]
          description: Состояние оплаты за услугу.

    AppointmentListResponse:
      type: object
      required:
        - total
        - limit
        - offset
        - data
      properties:
        total:
          type: integer
          description: Общее количество записей, подходящих под фильтр.
        limit:
          type: integer
          description: Количество записей, возвращённых в текущем ответе.
        offset:
          type: integer
          description: Смещение, использованное в запросе.
        data:
          type: array
          items:
            $ref: '#/components/schemas/Appointment'

    ErrorResponse:
      type: object
      required:
        - code
        - message
      properties:
        code:
          type: integer
          description: HTTP‑код ошибки.
        message:
          type: string
          description: Краткое описание причины ошибки.

```

---

### Краткое комментарий к выбранному подходу  

#### Пагинация  

* **Методы `limit` / `offset`** – простейший и привычный клиентам способ постраничного вывода.  
* `limit` ограничивает размер ответа (по умолчанию 20, максимум 100) → защищаем API от «троллей‑запросов», сразу удовлетворяя требование **Rate‑limiting**.  
* `offset` позволяет перейти к любой странице без необходимости хранить курсор‑состояние на сервере. При небольших объёмах (число записей у одного пациента < 10 000) производительность остаётся приемлемой, а реализация в репозитории PostgreSQL выглядит как обычный `SELECT … LIMIT … OFFSET …`.  

Если в дальнейшем понадобится «cursor‑based» пагинация (для масштабных списков, где `OFFSET` начинает сильно замедлять запрос), можно добавить отдельный endpoint `GET /appointments/cursor` – но на этапе MVP `limit/offset` полностью покрывает бизнес‑требования.

#### Версионирование API  

* **Версия в пути** – ` /api/v1/appointments`.  
* Такой подход прост для роутинга в API‑gateway (Kong/Envoy) и для автоматической генерации клиентских SDK.  
* При появлении несовместимых изменений будет создан новый путь `/api/v2/...`, а старый останется доступным минимум 6 месяцев, что упрощает миграцию мобильных клиентов.  

#### Безопасность  

* **Bearer‑JWT** в заголовке `Authorization`. Токен проверяется на уровне API‑gateway (поддерживает подпись RS256, проверку `exp`, `aud`, `iss`).  
* При отсутствии валидного токена сразу возвращается **401 Unauthorized** – клиент может инициировать повторный логин.  
* Все ответы (200/401/500) включают чётко описанные JSON‑payload’ы (`AppointmentListResponse` и `ErrorResponse`), что упрощает обработку ошибок в мобильном клиенте.  
