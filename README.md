# Short Polling
Пример написания sequence-диаграммы и OAS-контракта для Short-Polling.

Цель: Для некоторых систем появилась потребность получать статусы по заявке. В связи с этим необходимо разработать метод для опрашивания сервиса о статусе заявки. В нем нужно будет передавать идентификатор заявки, по которой будет запрашиваться статус. 

В ответ мы должны получать небольшой набор параметров:
* Идентификатор заявки;
* Тип заявки; 
* Дата создания;
* Статус.
  
На данном этапе статусная модель заявки будет состоять из:
* Принято;
* Рассматривается;
* Одобрено;
* Отклонено;
* Подтверждено.

Нефункциональные требования:
* В компании используются стандартные временные метки в формате ISO 8601 timestamp.
* Обычно в компании принято описывать ошибки 400, 404, 500.

## PlantUML 

``` plantuml
@startuml

actor User
participant Client
participant Server

User-> Client: отправить заявку
activate Client
Client -> Server: POST создание заявки
activate Server
Server --> Client: Статус = "Принято"
deactivate Server 
Client --> User: заявка создана
deactivate Client

loop Short-Polling пока статус не терминирующий
    Client -> Server: GET /application/status?applicationId=...
    activate Client 
    activate Server
    Server --> Client: ApplicationInfo (status, type)
    deactivate Server 
    opt статус != "Одобрено" или "Отклонено"
        Client --> User: обновление статуса
    end
    opt статус == "Одобрено" или "Отклонено"
        Client --> User: уведомление о финальном статусе
        deactivate Client
    end
end

@enduml
```
<img width="653" height="559" alt="image" src="https://github.com/user-attachments/assets/4bc7cf45-3150-4a56-8c9a-4681dcbf1eb7" />

## OpenApi Specification 

```
openapi: '3.0.3'
info:
  title: API Title
  version: '1.0'
servers:
  - url: https://api.server.test/v1
    description: Application server
paths:
  /application/status:
    get:
      summary: Получение статуса заявки
      description: Клиент отправляет запрос c id заявки до тех пор, пока не получит терминирующий статус.
      tags:
        - application
      parameters:
        - name: applicationId
          in: query
          schema:
            type: integer
            format: int64
          required: true
          description: ID заявки
      responses:
        '200':
          description: Успешный ответ с параметрами заявки
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ApplicationInfo'
        '400':
          $ref: '#/components/responses/BadRequest'
        '404':
          $ref: '#/components/responses/NotFound'
        '500':
          $ref: '#/components/responses/InternalServerError'
components:
  schemas:
    ApplicationInfo:
      type: object
      properties:
        applicationId:
          type: integer
          format: int64
          description: Уникальный идентификатор заявки
        type:
          type: string
          enum: 
            - Web
            - Mobile
            - Phone
          description: Тип заявки
        createDateTime:
          type: string
          format: date-time
          description: Дата-время создания заявки
        status:
          type: string
          enum: 
            - Принято
            - Рассматривается
            - Одобрено
            - Отклонено
            - Подтверждено
      required:
        - applicationId
        - type
        - createDateTime
        - status
      example:
        applicationId: 25692384
        type: "Web"
        createDateTime: "2025-04-05T12:34:56Z"
        status: "Принято"
    Error:
      type: object
      properties:
        code:
          type: integer
          maxLength: 999
        message:
          type: string
      required:
        - code
        - message
  responses:
    BadRequest:
      description: Bad request
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/Error'
          example:
            code: 400
            message: "Параметр applicationId должен быть положительным целым числом" 
    NotFound:
      description: Not found
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/Error'
          example:
            code: 404
            message: "Заявка с идентификатором 12345 не найдена"
    InternalServerError:
      description: Internal Server Error
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/Error'
          example:
            code: 500
            message: "Внутренняя ошибка сервера. Пожалуйста, попробуйте позже." 
```
