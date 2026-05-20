# OpenAPI Specification: Инвестиционная платформа (ЛК Брокера)

---

## Шаг 1 — Описание эндпоинтов текстом

### 1. Торговые заявки (Orders)

**POST /v1/orders** — Создать торговую заявку

Параметры тела (JSON):
- accountId (UUID, обязательный)
- instrumentId (UUID, обязательный)
- direction (enum: BUY / SELL)
- type (enum: MARKET / LIMIT)
- quantity (integer, >0)
- price (decimal, обязателен для LIMIT)

Заголовки: Authorization: Bearer JWT, Idempotency-Key

Бизнес-правила: сумма ≤ 10 000 000 руб., проверка средств, расчёт комиссии.

Ответы: 201, 400, 401, 403, 422 (LIMIT_EXCEEDED / INSUFFICIENT_FUNDS), 429, 500

---

**GET /v1/orders** — Список заявок пользователя

Query параметры: status, accountId, fromDate, toDate, limit, offset

Ответы: 200, 401, 403, 429, 500

---

**GET /v1/orders/{id}** — Детали заявки

Path: id (UUID)

Ответы: 200, 401, 403, 404, 429, 500

---

**DELETE /v1/orders/{id}** — Отменить заявку

Только из статуса PENDING или PROCESSING → CANCELLED

Ответы: 200, 400 (терминальный статус), 401, 403, 404, 429, 500

---

### 2. Портфель (Portfolio)

**GET /v1/portfolio** — Текущий портфель

Query: accountId, currency (RUB/USD/EUR/CNY)
Заголовки: If-None-Match для кэширования
Ответ включает ETag и Cache-Control.

Ответы: 200, 304 (Not Modified), 401, 403, 429, 500

---

**GET /v1/portfolio/performance** — Доходность портфеля

Query: period (day/week/month/year/all), accountId

Ответы: 200, 401, 403, 429, 500

---

### 3. Баланс и вывод средств

**GET /v1/accounts/{id}/balance** — Баланс счёта

Path: id (UUID счёта)

Ответ: { available, blocked, currency }

Ответы: 200, 401, 403, 404, 429, 500

---

**POST /v1/withdrawals** — Запросить вывод средств

Тело: accountId, amount, currency, bankDetails (bik, accountNumber, recipientName)

Заголовки: Idempotency-Key

Бизнес-правила: остаток ≥ amount, проверка KYC/AML

Ответы: 201, 400, 401, 403, 422 (INSUFFICIENT_FUNDS / COMPLIANCE_REJECTED), 429, 500

---

**GET /v1/withdrawals/{id}** — Статус заявки на вывод

Path: id (UUID)

Ответы: 200, 401, 403, 404, 429, 500

---

### 4. Транзакции (Transactions)

**GET /v1/transactions** — История транзакций

Query: accountId (обязательный), type, fromDate, toDate, limit, offset

Ответы: 200, 401, 403, 429, 500

---

**GET /v1/transactions/{id}** — Детали транзакции

Path: id (UUID)

Ответы: 200, 401, 403, 404, 429, 500

---

### Общие коды ответов

| Код | Описание |
|---|---|
| 200 / 201 | Успех |
| 400 | Невалидный запрос |
| 401 | Не авторизован |
| 403 | Нет прав |
| 404 | Не найдено |
| 422 | Бизнес-ошибка |
| 429 | Rate limit (заголовок Retry-After) |
| 500 | Внутренняя ошибка |

---

## Шаг 2 — YAML (OpenAPI 3.0.3)

Скопируй YAML ниже и вставь на **editor.swagger.io** для валидации.

```yaml
openapi: "3.0.3"
info:
  title: Broker Platform API
  description: API для Личного кабинета брокера — торговые заявки, портфель, вывод средств, транзакции
  version: "1.0.0"
  contact:
    name: Support Team
    email: support@broker.ru
servers:
  - url: https://api.broker.ru/v1
    description: Production server
  - url: https://sandbox-api.broker.ru/v1
    description: Sandbox server

security:
  - BearerAuth: []

tags:
  - name: Orders
    description: Управление торговыми заявками
  - name: Portfolio
    description: Портфель и доходность
  - name: Withdrawals
    description: Вывод средств
  - name: Transactions
    description: История транзакций
  - name: Accounts
    description: Счета и балансы

paths:
  /orders:
    post:
      tags: [Orders]
      summary: Создать торговую заявку
      description: Подача рыночной или лимитной заявки с проверкой лимита 10 000 000 руб.
      operationId: createOrder
      parameters:
        - $ref: '#/components/parameters/IdempotencyKey'
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/OrderRequest'
            examples:
              market:
                summary: Рыночная заявка на покупку
                value:
                  accountId: "3fa85f64-5717-4562-b3fc-2c963f66afa6"
                  instrumentId: "1a2b3c4d-5e6f-7a8b-9c0d-1e2f3a4b5c6d"
                  direction: "BUY"
                  type: "MARKET"
                  quantity: 10
              limit:
                summary: Лимитная заявка на продажу
                value:
                  accountId: "3fa85f64-5717-4562-b3fc-2c963f66afa6"
                  instrumentId: "2b3c4d5e-6f7a-8b9c-0d1e-2f3a4b5c6d7e"
                  direction: "SELL"
                  type: "LIMIT"
                  quantity: 5
                  price: 150.50
      responses:
        '201':
          description: Заявка успешно создана
          headers:
            ETag:
              schema:
                type: string
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/OrderResponse'
              example:
                id: "f47ac10b-58cc-4372-a567-0e02b2c3d479"
                status: "PENDING"
                commission:
                  amount: 12.50
                  type: "PERCENTAGE"
                createdAt: "2025-01-27T10:30:00Z"
        '400':
          $ref: '#/components/responses/BadRequest'
        '401':
          $ref: '#/components/responses/Unauthorized'
        '403':
          $ref: '#/components/responses/Forbidden'
        '422':
          description: Бизнес-ошибка
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Error'
              examples:
                limitExceeded:
                  value:
                    code: "LIMIT_EXCEEDED"
                    message: "Сумма заявки превышает максимально допустимую (10 000 000 руб.)"
                insufficientFunds:
                  value:
                    code: "INSUFFICIENT_FUNDS"
                    message: "Недостаточно средств на счёте"
        '429':
          $ref: '#/components/responses/TooManyRequests'
        '500':
          $ref: '#/components/responses/InternalError'
    get:
      tags: [Orders]
      summary: Получить список заявок
      operationId: getOrders
      parameters:
        - name: status
          in: query
          schema:
            type: string
            enum: [PENDING, PROCESSING, COMPLETED, FAILED, CANCELLED]
        - name: accountId
          in: query
          schema:
            type: string
            format: uuid
        - name: fromDate
          in: query
          schema:
            type: string
            format: date-time
        - name: toDate
          in: query
          schema:
            type: string
            format: date-time
        - name: limit
          in: query
          schema:
            type: integer
            default: 20
            minimum: 1
            maximum: 100
        - name: offset
          in: query
          schema:
            type: integer
            default: 0
      responses:
        '200':
          description: Список заявок
          content:
            application/json:
              schema:
                type: object
                properties:
                  items:
                    type: array
                    items:
                      $ref: '#/components/schemas/OrderShort'
                  total:
                    type: integer
                  limit:
                    type: integer
                  offset:
                    type: integer
        '401':
          $ref: '#/components/responses/Unauthorized'
        '403':
          $ref: '#/components/responses/Forbidden'
        '429':
          $ref: '#/components/responses/TooManyRequests'
        '500':
          $ref: '#/components/responses/InternalError'

  /orders/{id}:
    get:
      tags: [Orders]
      summary: Получить заявку по ID
      operationId: getOrderById
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: string
            format: uuid
      responses:
        '200':
          description: Детальная информация о заявке
          headers:
            ETag:
              schema:
                type: string
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/OrderDetails'
        '401':
          $ref: '#/components/responses/Unauthorized'
        '403':
          $ref: '#/components/responses/Forbidden'
        '404':
          $ref: '#/components/responses/NotFound'
        '429':
          $ref: '#/components/responses/TooManyRequests'
        '500':
          $ref: '#/components/responses/InternalError'
    delete:
      tags: [Orders]
      summary: Отменить заявку
      operationId: cancelOrder
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: string
            format: uuid
      responses:
        '200':
          description: Заявка отменена
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/OrderResponse'
              example:
                id: "f47ac10b-58cc-4372-a567-0e02b2c3d479"
                status: "CANCELLED"
        '400':
          description: Невозможно отменить (терминальный статус)
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Error'
        '401':
          $ref: '#/components/responses/Unauthorized'
        '403':
          $ref: '#/components/responses/Forbidden'
        '404':
          $ref: '#/components/responses/NotFound'
        '429':
          $ref: '#/components/responses/TooManyRequests'
        '500':
          $ref: '#/components/responses/InternalError'

  /portfolio:
    get:
      tags: [Portfolio]
      summary: Получить портфель
      operationId: getPortfolio
      parameters:
        - name: accountId
          in: query
          schema:
            type: string
            format: uuid
        - name: currency
          in: query
          schema:
            type: string
            enum: [RUB, USD, EUR, CNY]
            default: RUB
        - name: If-None-Match
          in: header
          schema:
            type: string
          description: ETag из предыдущего ответа
      responses:
        '200':
          description: Портфель
          headers:
            ETag:
              schema:
                type: string
            Cache-Control:
              schema:
                type: string
              example: "private, max-age=60"
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Portfolio'
        '304':
          description: Не изменялся (кэш актуален)
        '401':
          $ref: '#/components/responses/Unauthorized'
        '403':
          $ref: '#/components/responses/Forbidden'
        '429':
          $ref: '#/components/responses/TooManyRequests'
        '500':
          $ref: '#/components/responses/InternalError'

  /portfolio/performance:
    get:
      tags: [Portfolio]
      summary: Доходность портфеля
      operationId: getPortfolioPerformance
      parameters:
        - name: period
          in: query
          schema:
            type: string
            enum: [day, week, month, year, all]
            default: month
        - name: accountId
          in: query
          schema:
            type: string
            format: uuid
      responses:
        '200':
          description: Массив точек доходности
          content:
            application/json:
              schema:
                type: array
                items:
                  $ref: '#/components/schemas/PerformancePoint'
        '401':
          $ref: '#/components/responses/Unauthorized'
        '403':
          $ref: '#/components/responses/Forbidden'
        '429':
          $ref: '#/components/responses/TooManyRequests'
        '500':
          $ref: '#/components/responses/InternalError'

  /accounts/{id}/balance:
    get:
      tags: [Accounts]
      summary: Баланс счёта
      operationId: getAccountBalance
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: string
            format: uuid
      responses:
        '200':
          description: Баланс и заблокированные средства
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Balance'
        '401':
          $ref: '#/components/responses/Unauthorized'
        '403':
          $ref: '#/components/responses/Forbidden'
        '404':
          $ref: '#/components/responses/NotFound'
        '429':
          $ref: '#/components/responses/TooManyRequests'
        '500':
          $ref: '#/components/responses/InternalError'

  /withdrawals:
    post:
      tags: [Withdrawals]
      summary: Запросить вывод средств
      operationId: createWithdrawal
      parameters:
        - $ref: '#/components/parameters/IdempotencyKey'
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/WithdrawalRequest'
      responses:
        '201':
          description: Заявка на вывод принята
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/WithdrawalResponse'
        '400':
          $ref: '#/components/responses/BadRequest'
        '401':
          $ref: '#/components/responses/Unauthorized'
        '403':
          $ref: '#/components/responses/Forbidden'
        '422':
          description: Бизнес-ошибка
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Error'
              examples:
                insufficientFunds:
                  value:
                    code: "INSUFFICIENT_FUNDS"
                    message: "Недостаточно средств"
                complianceRejected:
                  value:
                    code: "COMPLIANCE_REJECTED"
                    message: "Вывод отклонён службой комплаенс"
        '429':
          $ref: '#/components/responses/TooManyRequests'
        '500':
          $ref: '#/components/responses/InternalError'

  /withdrawals/{id}:
    get:
      tags: [Withdrawals]
      summary: Статус заявки на вывод
      operationId: getWithdrawalStatus
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: string
            format: uuid
      responses:
        '200':
          description: Информация о выводе
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/WithdrawalDetails'
        '401':
          $ref: '#/components/responses/Unauthorized'
        '403':
          $ref: '#/components/responses/Forbidden'
        '404':
          $ref: '#/components/responses/NotFound'
        '429':
          $ref: '#/components/responses/TooManyRequests'
        '500':
          $ref: '#/components/responses/InternalError'

  /transactions:
    get:
      tags: [Transactions]
      summary: История транзакций
      operationId: getTransactions
      parameters:
        - name: accountId
          in: query
          required: true
          schema:
            type: string
            format: uuid
        - name: type
          in: query
          schema:
            type: string
            enum: [ORDER_EXECUTION, WITHDRAWAL, DEPOSIT, COMMISSION]
        - name: fromDate
          in: query
          schema:
            type: string
            format: date-time
        - name: toDate
          in: query
          schema:
            type: string
            format: date-time
        - name: limit
          in: query
          schema:
            type: integer
            default: 20
        - name: offset
          in: query
          schema:
            type: integer
            default: 0
      responses:
        '200':
          description: Список транзакций
          content:
            application/json:
              schema:
                type: object
                properties:
                  items:
                    type: array
                    items:
                      $ref: '#/components/schemas/TransactionShort'
                  total:
                    type: integer
        '401':
          $ref: '#/components/responses/Unauthorized'
        '403':
          $ref: '#/components/responses/Forbidden'
        '429':
          $ref: '#/components/responses/TooManyRequests'
        '500':
          $ref: '#/components/responses/InternalError'

  /transactions/{id}:
    get:
      tags: [Transactions]
      summary: Детали транзакции
      operationId: getTransactionById
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: string
            format: uuid
      responses:
        '200':
          description: Полная информация о транзакции
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/TransactionDetails'
        '401':
          $ref: '#/components/responses/Unauthorized'
        '403':
          $ref: '#/components/responses/Forbidden'
        '404':
          $ref: '#/components/responses/NotFound'
        '429':
          $ref: '#/components/responses/TooManyRequests'
        '500':
          $ref: '#/components/responses/InternalError'

components:
  securitySchemes:
    BearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
      description: JWT токен, полученный при аутентификации

  parameters:
    IdempotencyKey:
      name: Idempotency-Key
      in: header
      required: true
      schema:
        type: string
        format: uuid
      description: Уникальный ключ идемпотентности для POST запросов

  responses:
    BadRequest:
      description: Невалидный запрос
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/Error'
          example:
            code: "BAD_REQUEST"
            message: "Некорректный JSON или отсутствуют обязательные поля"
    Unauthorized:
      description: Не авторизован
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/Error'
          example:
            code: "UNAUTHORIZED"
            message: "Требуется аутентификация"
    Forbidden:
      description: Доступ запрещён
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/Error'
          example:
            code: "FORBIDDEN"
            message: "Нет прав на выполнение операции"
    NotFound:
      description: Ресурс не найден
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/Error'
          example:
            code: "NOT_FOUND"
            message: "Заявка не найдена"
    TooManyRequests:
      description: Превышен лимит запросов
      headers:
        Retry-After:
          schema:
            type: integer
          description: Рекомендуемое время ожидания в секундах
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/Error'
          example:
            code: "RATE_LIMIT_EXCEEDED"
            message: "Слишком много запросов. Повторите через 30 секунд"
    InternalError:
      description: Внутренняя ошибка сервера
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/Error'
          example:
            code: "INTERNAL_ERROR"
            message: "Произошла ошибка на сервере"

  schemas:
    Error:
      type: object
      properties:
        code:
          type: string
        message:
          type: string
      required: [code, message]

    OrderRequest:
      type: object
      properties:
        accountId:
          type: string
          format: uuid
        instrumentId:
          type: string
          format: uuid
        direction:
          type: string
          enum: [BUY, SELL]
        type:
          type: string
          enum: [MARKET, LIMIT]
        quantity:
          type: integer
          minimum: 1
        price:
          type: number
          format: decimal
          description: Обязателен для LIMIT, игнорируется для MARKET
      required: [accountId, instrumentId, direction, type, quantity]

    OrderResponse:
      type: object
      properties:
        id:
          type: string
          format: uuid
        status:
          type: string
          enum: [PENDING, PROCESSING, COMPLETED, FAILED, CANCELLED]
        commission:
          type: object
          properties:
            amount:
              type: number
              format: decimal
            type:
              type: string
              enum: [FIXED, PERCENTAGE, MIXED]
        createdAt:
          type: string
          format: date-time

    OrderShort:
      type: object
      properties:
        id:
          type: string
          format: uuid
        direction:
          type: string
        type:
          type: string
        quantity:
          type: integer
        status:
          type: string
        createdAt:
          type: string
          format: date-time

    OrderDetails:
      allOf:
        - $ref: '#/components/schemas/OrderResponse'
        - type: object
          properties:
            accountId:
              type: string
              format: uuid
            instrumentId:
              type: string
              format: uuid
            amount:
              type: number
              format: decimal
            price:
              type: number
              format: decimal
            executedAt:
              type: string
              format: date-time
            failureReason:
              type: string

    Portfolio:
      type: object
      properties:
        totalValue:
          type: number
          format: decimal
          description: Общая стоимость портфеля в выбранной валюте
        freeBalance:
          type: number
          format: decimal
        positions:
          type: array
          items:
            type: object
            properties:
              instrumentId:
                type: string
                format: uuid
              symbol:
                type: string
              name:
                type: string
              quantity:
                type: number
                format: decimal
              avgPrice:
                type: number
                format: decimal
              currentPrice:
                type: number
                format: decimal
              currentValue:
                type: number
                format: decimal
              profitPercent:
                type: number
                format: decimal
              currency:
                type: string

    PerformancePoint:
      type: object
      properties:
        date:
          type: string
          format: date
        totalValue:
          type: number
          format: decimal
        profitPercent:
          type: number
          format: decimal

    Balance:
      type: object
      properties:
        available:
          type: number
          format: decimal
        blocked:
          type: number
          format: decimal
        currency:
          type: string
          enum: [RUB, USD, EUR, CNY]

    WithdrawalRequest:
      type: object
      properties:
        accountId:
          type: string
          format: uuid
        amount:
          type: number
          format: decimal
          minimum: 0.01
        currency:
          type: string
          enum: [RUB, USD, EUR, CNY]
        bankDetails:
          type: object
          properties:
            bik:
              type: string
            accountNumber:
              type: string
            recipientName:
              type: string
          required: [bik, accountNumber, recipientName]
      required: [accountId, amount, currency, bankDetails]

    WithdrawalResponse:
      type: object
      properties:
        id:
          type: string
          format: uuid
        status:
          type: string
          enum: [PENDING, PROCESSING, COMPLETED, FAILED, CANCELLED]
        createdAt:
          type: string
          format: date-time

    WithdrawalDetails:
      allOf:
        - $ref: '#/components/schemas/WithdrawalResponse'
        - type: object
          properties:
            amount:
              type: number
              format: decimal
            currency:
              type: string
            complianceStatus:
              type: string
              enum: [OK, REJECTED]
            processedAt:
              type: string
              format: date-time
            failureReason:
              type: string

    TransactionShort:
      type: object
      properties:
        id:
          type: string
          format: uuid
        type:
          type: string
        amount:
          type: number
          format: decimal
        currency:
          type: string
        createdAt:
          type: string
          format: date-time

    TransactionDetails:
      allOf:
        - $ref: '#/components/schemas/TransactionShort'
        - type: object
          properties:
            status:
              type: string
              enum: [PENDING, COMPLETED, FAILED]
            description:
              type: string
            orderId:
              type: string
              format: uuid
            withdrawalId:
              type: string
              format: uuid
```

---

> **Инструкция по валидации:**
> 1. Открой [editor.swagger.io](https://editor.swagger.io)
> 2. Выдели весь текст слева и удали его
> 3. Вставь YAML из блока выше
> 4. Убедись что справа отображается документация без ошибок
> 5. Сохрани скриншот для подтверждения валидации
