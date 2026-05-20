# ER-диаграмма: Инвестиционная платформа (ЛК Брокера)

---

## Шаг 1 — Описание сущностей текстом

### 1. users

```
PK  id              UUID
    email           VARCHAR(255) NOT NULL UNIQUE
    phone           VARCHAR(20)
    full_name       VARCHAR(255)
    kyc_status      VARCHAR(50)   — VERIFIED, PENDING, REJECTED
    aml_flag        BOOLEAN
    created_at      TIMESTAMP
    updated_at      TIMESTAMP
```

### 2. accounts

```
PK  id              UUID
FK  user_id         UUID → users(id)
    account_number  VARCHAR(50) UNIQUE
    currency        VARCHAR(3)    — RUB, USD, EUR, CNY
    balance         DECIMAL(18,2)
    blocked_amount  DECIMAL(18,2) — средства в заявках/выводах
    status          VARCHAR(50)   — ACTIVE, BLOCKED, CLOSED
    created_at      TIMESTAMP
```

### 3. instruments

```
PK  id              UUID
    symbol          VARCHAR(20) UNIQUE
    name            VARCHAR(255)
    instrument_type VARCHAR(50)   — STOCK, BOND, ETF, FUTURES
    currency        VARCHAR(3)
    lot_size        INTEGER
    is_tradable     BOOLEAN
```

### 4. orders

```
PK  id                UUID
FK  account_id        UUID → accounts(id)
FK  instrument_id     UUID → instruments(id)
    type              VARCHAR(50)   — MARKET, LIMIT
    direction         VARCHAR(10)   — BUY, SELL
    quantity          INTEGER
    price             DECIMAL(18,2) — NULL для рыночной заявки
    amount            DECIMAL(18,2) — quantity × price
    commission_amount DECIMAL(18,2)
    commission_type   VARCHAR(50)   — FIXED, PERCENTAGE, MIXED
    status            VARCHAR(50)   — PENDING, PROCESSING,
                                       COMPLETED, FAILED, CANCELLED
    idempotency_key   VARCHAR(255)
    created_at        TIMESTAMP
    updated_at        TIMESTAMP
    executed_at       TIMESTAMP
```

### 5. transactions

```
PK  id          UUID
FK  account_id  UUID → accounts(id)
FK  order_id    UUID → orders(id) NULL
    type        VARCHAR(50)   — ORDER_EXECUTION, WITHDRAWAL,
                                 DEPOSIT, COMMISSION
    amount      DECIMAL(18,2)
    currency    VARCHAR(3)
    status      VARCHAR(50)   — PENDING, COMPLETED, FAILED
    description TEXT
    created_at  TIMESTAMP
```

### 6. portfolios

```
PK  id              UUID
FK  account_id      UUID → accounts(id)
FK  instrument_id   UUID → instruments(id)
    quantity        DECIMAL(18,4) — текущее количество
    avg_price       DECIMAL(18,2) — средняя цена входа
    current_price   DECIMAL(18,2) — последняя известная
    updated_at      TIMESTAMP
```

### 7. commission_rules

```
PK  id              UUID
    instrument_type VARCHAR(50)    — STOCK, BOND, ETF, FUTURES
    commission_type VARCHAR(50)    — FIXED, PERCENTAGE, MIXED
    value           DECIMAL(10,4)  — для FIXED: сумма;
                                     для PERCENTAGE: проценты
    value2          DECIMAL(10,4)  — для MIXED: доп. часть
    min_commission  DECIMAL(18,2)
    max_commission  DECIMAL(18,2)
    valid_from      TIMESTAMP
    valid_to        TIMESTAMP
```

### 8. withdrawal_requests

```
PK  id                  UUID
FK  account_id          UUID → accounts(id)
    amount              DECIMAL(18,2)
    currency            VARCHAR(3)
    bank_details        TEXT          — зашифрованные реквизиты
    status              VARCHAR(50)   — PENDING, PROCESSING,
                                         COMPLETED, FAILED, CANCELLED
    compliance_status   VARCHAR(50)   — OK, REJECTED
    payment_gateway_id  VARCHAR(255)
    created_at          TIMESTAMP
    processed_at        TIMESTAMP
```

### 9. notifications

```
PK  id          UUID
FK  user_id     UUID → users(id)
    type        VARCHAR(50)   — EMAIL, PUSH
    channel     VARCHAR(50)   — ORDER_STATUS, WITHDRAWAL_STATUS,
                                 PORTFOLIO_ALERT
    subject     VARCHAR(255)
    content     TEXT
    status      VARCHAR(50)   — PENDING, SENT, FAILED
    created_at  TIMESTAMP
    sent_at     TIMESTAMP
```

---

### Связи (1 — N)

| Таблица А | Тип | Таблица Б | Пояснение |
|---|---|---|---|
| users | 1 — N | accounts | Один пользователь — несколько счетов (разные валюты) |
| accounts | 1 — N | orders | Один счёт — много заявок |
| instruments | 1 — N | orders | Один инструмент — во многих заявках |
| accounts | 1 — N | transactions | Один счёт — много транзакций |
| orders | 1 — 0..1 | transactions | Заявка может породить одну транзакцию |
| accounts | 1 — N | portfolios | Один счёт — позиции по разным инструментам |
| instruments | 1 — N | portfolios | Один инструмент — в портфелях разных счетов |
| commission_rules | 1 — N | orders | Одно правило — много заявок (через instrument_type) |
| accounts | 1 — N | withdrawal_requests | Один счёт — много заявок на вывод |
| users | 1 — N | notifications | Один пользователь — много уведомлений |

---

## Шаг 2 — XML для draw.io

Скопируй XML ниже полностью.
В draw.io: **Extras → Edit Diagram → вставь XML → OK**

```xml
<mxfile host="app.diagrams.net" modified="2025-01-27T00:00:00.000Z" agent="ERD Generator" version="22.1.0" etag="erd-broker" type="device">
  <diagram name="ER-диаграмма брокерской платформы" id="erd-broker">
    <mxGraphModel dx="1200" dy="800" grid="1" gridSize="10" guides="1" tooltips="1" connect="1" arrows="1" fold="1" page="1" pageScale="1" pageWidth="1600" pageHeight="1200" math="0" shadow="0">
      <root>
        <mxCell id="0"/>
        <mxCell id="1" parent="0"/>
        <mxCell id="users" value="users" style="shape=table;startSize=30;container=1;collapsible=0;childLayout=tableLayout;fixedRows=1;rowLines=0;fontStyle=1;align=center;resizeLast=1;" vertex="1" parent="1"><mxGeometry x="40" y="40" width="260" height="250" as="geometry"/></mxCell>
        <mxCell id="users_pk" value="PK  id                UUID" style="shape=partialRectangle;top=0;left=0;right=0;bottom=0;align=left;verticalAlign=top;spacingLeft=6;spacingRight=6;overflow=hidden;whiteSpace=wrap;" vertex="1" parent="users"><mxGeometry x="10" y="30" width="240" height="20" as="geometry"/></mxCell>
        <mxCell id="users_email" value="    email             VARCHAR(255) NOT NULL" style="shape=partialRectangle;top=0;left=0;right=0;bottom=0;align=left;verticalAlign=top;spacingLeft=6;spacingRight=6;overflow=hidden;whiteSpace=wrap;" vertex="1" parent="users"><mxGeometry x="10" y="50" width="240" height="20" as="geometry"/></mxCell>
        <mxCell id="users_phone" value="    phone             VARCHAR(20)" style="shape=partialRectangle;top=0;left=0;right=0;bottom=0;align=left;verticalAlign=top;spacingLeft=6;spacingRight=6;overflow=hidden;whiteSpace=wrap;" vertex="1" parent="users"><mxGeometry x="10" y="70" width="240" height="20" as="geometry"/></mxCell>
        <mxCell id="users_name" value="    full_name         VARCHAR(255)" style="shape=partialRectangle;top=0;left=0;right=0;bottom=0;align=left;verticalAlign=top;spacingLeft=6;spacingRight=6;overflow=hidden;whiteSpace=wrap;" vertex="1" parent="users"><mxGeometry x="10" y="90" width="240" height="20" as="geometry"/></mxCell>
        <mxCell id="users_kyc" value="    kyc_status        VARCHAR(50)" style="shape=partialRectangle;top=0;left=0;right=0;bottom=0;align=left;verticalAlign=top;spacingLeft=6;spacingRight=6;overflow=hidden;whiteSpace=wrap;" vertex="1" parent="users"><mxGeometry x="10" y="110" width="240" height="20" as="geometry"/></mxCell>
        <mxCell id="users_aml" value="    aml_flag          BOOLEAN" style="shape=partialRectangle;top=0;left=0;right=0;bottom=0;align=left;verticalAlign=top;spacingLeft=6;spacingRight=6;overflow=hidden;whiteSpace=wrap;" vertex="1" parent="users"><mxGeometry x="10" y="130" width="240" height="20" as="geometry"/></mxCell>
        <mxCell id="users_created" value="    created_at        TIMESTAMP" style="shape=partialRectangle;top=0;left=0;right=0;bottom=0;align=left;verticalAlign=top;spacingLeft=6;spacingRight=6;overflow=hidden;whiteSpace=wrap;" vertex="1" parent="users"><mxGeometry x="10" y="150" width="240" height="20" as="geometry"/></mxCell>
        <mxCell id="users_updated" value="    updated_at        TIMESTAMP" style="shape=partialRectangle;top=0;left=0;right=0;bottom=0;align=left;verticalAlign=top;spacingLeft=6;spacingRight=6;overflow=hidden;whiteSpace=wrap;" vertex="1" parent="users"><mxGeometry x="10" y="170" width="240" height="20" as="geometry"/></mxCell>
        <mxCell id="accounts" value="accounts" style="shape=table;startSize=30;container=1;collapsible=0;childLayout=tableLayout;fixedRows=1;rowLines=0;fontStyle=1;align=center;resizeLast=1;" vertex="1" parent="1"><mxGeometry x="400" y="40" width="280" height="250" as="geometry"/></mxCell>
        <mxCell id="accounts_pk" value="PK  id                UUID" style="shape=partialRectangle;top=0;left=0;right=0;bottom=0;align=left;verticalAlign=top;spacingLeft=6;spacingRight=6;overflow=hidden;whiteSpace=wrap;" vertex="1" parent="accounts"><mxGeometry x="10" y="30" width="260" height="20" as="geometry"/></mxCell>
        <mxCell id="accounts_fk" value="FK  user_id           UUID" style="shape=partialRectangle;top=0;left=0;right=0;bottom=0;align=left;verticalAlign=top;spacingLeft=6;spacingRight=6;overflow=hidden;whiteSpace=wrap;" vertex="1" parent="accounts"><mxGeometry x="10" y="50" width="260" height="20" as="geometry"/></mxCell>
        <mxCell id="accounts_num" value="    account_number    VARCHAR(50) UNIQUE" style="shape=partialRectangle;top=0;left=0;right=0;bottom=0;align=left;verticalAlign=top;spacingLeft=6;spacingRight=6;overflow=hidden;whiteSpace=wrap;" vertex="1" parent="accounts"><mxGeometry x="10" y="70" width="260" height="20" as="geometry"/></mxCell>
        <mxCell id="accounts_curr" value="    currency          VARCHAR(3)" style="shape=partialRectangle;top=0;left=0;right=0;bottom=0;align=left;verticalAlign=top;spacingLeft=6;spacingRight=6;overflow=hidden;whiteSpace=wrap;" vertex="1" parent="accounts"><mxGeometry x="10" y="90" width="260" height="20" as="geometry"/></mxCell>
        <mxCell id="accounts_bal" value="    balance           DECIMAL(18,2)" style="shape=partialRectangle;top=0;left=0;right=0;bottom=0;align=left;verticalAlign=top;spacingLeft=6;spacingRight=6;overflow=hidden;whiteSpace=wrap;" vertex="1" parent="accounts"><mxGeometry x="10" y="110" width="260" height="20" as="geometry"/></mxCell>
        <mxCell id="accounts_blocked" value="    blocked_amount    DECIMAL(18,2)" style="shape=partialRectangle;top=0;left=0;right=0;bottom=0;align=left;verticalAlign=top;spacingLeft=6;spacingRight=6;overflow=hidden;whiteSpace=wrap;" vertex="1" parent="accounts"><mxGeometry x="10" y="130" width="260" height="20" as="geometry"/></mxCell>
        <mxCell id="accounts_status" value="    status            VARCHAR(50)" style="shape=partialRectangle;top=0;left=0;right=0;bottom=0;align=left;verticalAlign=top;spacingLeft=6;spacingRight=6;overflow=hidden;whiteSpace=wrap;" vertex="1" parent="accounts"><mxGeometry x="10" y="150" width="260" height="20" as="geometry"/></mxCell>
        <mxCell id="accounts_created" value="    created_at        TIMESTAMP" style="shape=partialRectangle;top=0;left=0;right=0;bottom=0;align=left;verticalAlign=top;spacingLeft=6;spacingRight=6;overflow=hidden;whiteSpace=wrap;" vertex="1" parent="accounts"><mxGeometry x="10" y="170" width="260" height="20" as="geometry"/></mxCell>
        <mxCell id="instruments" value="instruments" style="shape=table;startSize=30;container=1;collapsible=0;childLayout=tableLayout;fixedRows=1;rowLines=0;fontStyle=1;align=center;resizeLast=1;" vertex="1" parent="1"><mxGeometry x="800" y="40" width="280" height="230" as="geometry"/></mxCell>
        <mxCell id="inst_pk" value="PK  id                UUID" style="shape=partialRectangle;top=0;left=0;right=0;bottom=0;align=left;verticalAlign=top;spacingLeft=6;spacingRight=6;overflow=hidden;whiteSpace=wrap;" vertex="1" parent="instruments"><mxGeometry x="10" y="30" width="260" height="20" as="geometry"/></mxCell>
        <mxCell id="inst_symbol" value="    symbol            VARCHAR(20) UNIQUE" style="shape=partialRectangle;top=0;left=0;right=0;bottom=0;align=left;verticalAlign=top;spacingLeft=6;spacingRight=6;overflow=hidden;whiteSpace=wrap;" vertex="1" parent="instruments"><mxGeometry x="10" y="50" width="260" height="20" as="geometry"/></mxCell>
        <mxCell id="inst_name" value="    name              VARCHAR(255)" style="shape=partialRectangle;top=0;left=0;right=0;bottom=0;align=left;verticalAlign=top;spacingLeft=6;spacingRight=6;overflow=hidden;whiteSpace=wrap;" vertex="1" parent="instruments"><mxGeometry x="10" y="70" width="260" height="20" as="geometry"/></mxCell>
        <mxCell id="inst_type" value="    instrument_type   VARCHAR(50)" style="shape=partialRectangle;top=0;left=0;right=0;bottom=0;align=left;verticalAlign=top;spacingLeft=6;spacingRight=6;overflow=hidden;whiteSpace=wrap;" vertex="1" parent="instruments"><mxGeometry x="10" y="90" width="260" height="20" as="geometry"/></mxCell>
        <mxCell id="inst_curr" value="    currency          VARCHAR(3)" style="shape=partialRectangle;top=0;left=0;right=0;bottom=0;align=left;verticalAlign=top;spacingLeft=6;spacingRight=6;overflow=hidden;whiteSpace=wrap;" vertex="1" parent="instruments"><mxGeometry x="10" y="110" width="260" height="20" as="geometry"/></mxCell>
        <mxCell id="inst_lot" value="    lot_size          INTEGER" style="shape=partialRectangle;top=0;left=0;right=0;bottom=0;align=left;verticalAlign=top;spacingLeft=6;spacingRight=6;overflow=hidden;whiteSpace=wrap;" vertex="1" parent="instruments"><mxGeometry x="10" y="130" width="260" height="20" as="geometry"/></mxCell>
        <mxCell id="inst_tradable" value="    is_tradable       BOOLEAN" style="shape=partialRectangle;top=0;left=0;right=0;bottom=0;align=left;verticalAlign=top;spacingLeft=6;spacingRight=6;overflow=hidden;whiteSpace=wrap;" vertex="1" parent="instruments"><mxGeometry x="10" y="150" width="260" height="20" as="geometry"/></mxCell>
        <mxCell id="orders" value="orders" style="shape=table;startSize=30;container=1;collapsible=0;childLayout=tableLayout;fixedRows=1;rowLines=0;fontStyle=1;align=center;resizeLast=1;" vertex="1" parent="1"><mxGeometry x="40" y="360" width="320" height="330" as="geometry"/></mxCell>
        <mxCell id="orders_pk" value="PK  id                UUID" style="shape=partialRectangle;top=0;left=0;right=0;bottom=0;align=left;verticalAlign=top;spacingLeft=6;spacingRight=6;overflow=hidden;whiteSpace=wrap;" vertex="1" parent="orders"><mxGeometry x="10" y="30" width="300" height="20" as="geometry"/></mxCell>
        <mxCell id="orders_fk_acc" value="FK  account_id        UUID" style="shape=partialRectangle;top=0;left=0;right=0;bottom=0;align=left;verticalAlign=top;spacingLeft=6;spacingRight=6;overflow=hidden;whiteSpace=wrap;" vertex="1" parent="orders"><mxGeometry x="10" y="50" width="300" height="20" as="geometry"/></mxCell>
        <mxCell id="orders_fk_inst" value="FK  instrument_id     UUID" style="shape=partialRectangle;top=0;left=0;right=0;bottom=0;align=left;verticalAlign=top;spacingLeft=6;spacingRight=6;overflow=hidden;whiteSpace=wrap;" vertex="1" parent="orders"><mxGeometry x="10" y="70" width="300" height="20" as="geometry"/></mxCell>
        <mxCell id="orders_type" value="    type              VARCHAR(50)" style="shape=partialRectangle;top=0;left=0;right=0;bottom=0;align=left;verticalAlign=top;spacingLeft=6;spacingRight=6;overflow=hidden;whiteSpace=wrap;" vertex="1" parent="orders"><mxGeometry x="10" y="90" width="300" height="20" as="geometry"/></mxCell>
        <mxCell id="orders_dir" value="    direction         VARCHAR(10)" style="shape=partialRectangle;top=0;left=0;right=0;bottom=0;align=left;verticalAlign=top;spacingLeft=6;spacingRight=6;overflow=hidden;whiteSpace=wrap;" vertex="1" parent="orders"><mxGeometry x="10" y="110" width="300" height="20" as="geometry"/></mxCell>
        <mxCell id="orders_qty" value="    quantity          INTEGER" style="shape=partialRectangle;top=0;left=0;right=0;bottom=0;align=left;verticalAlign=top;spacingLeft=6;spacingRight=6;overflow=hidden;whiteSpace=wrap;" vertex="1" parent="orders"><mxGeometry x="10" y="130" width="300" height="20" as="geometry"/></mxCell>
        <mxCell id="orders_price" value="    price             DECIMAL(18,2)" style="shape=partialRectangle;top=0;left=0;right=0;bottom=0;align=left;verticalAlign=top;spacingLeft=6;spacingRight=6;overflow=hidden;whiteSpace=wrap;" vertex="1" parent="orders"><mxGeometry x="10" y="150" width="300" height="20" as="geometry"/></mxCell>
        <mxCell id="orders_amount" value="    amount            DECIMAL(18,2)" style="shape=partialRectangle;top=0;left=0;right=0;bottom=0;align=left;verticalAlign=top;spacingLeft=6;spacingRight=6;overflow=hidden;whiteSpace=wrap;" vertex="1" parent="orders"><mxGeometry x="10" y="170" width="300" height="20" as="geometry"/></mxCell>
        <mxCell id="orders_comm" value="    commission_amount DECIMAL(18,2)" style="shape=partialRectangle;top=0;left=0;right=0;bottom=0;align=left;verticalAlign=top;spacingLeft=6;spacingRight=6;overflow=hidden;whiteSpace=wrap;" vertex="1" parent="orders"><mxGeometry x="10" y="190" width="300" height="20" as="geometry"/></mxCell>
        <mxCell id="orders_status" value="    status            VARCHAR(50)" style="shape=partialRectangle;top=0;left=0;right=0;bottom=0;align=left;verticalAlign=top;spacingLeft=6;spacingRight=6;overflow=hidden;whiteSpace=wrap;" vertex="1" parent="orders"><mxGeometry x="10" y="210" width="300" height="20" as="geometry"/></mxCell>
        <mxCell id="orders_idem" value="    idempotency_key   VARCHAR(255)" style="shape=partialRectangle;top=0;left=0;right=0;bottom=0;align=left;verticalAlign=top;spacingLeft=6;spacingRight=6;overflow=hidden;whiteSpace=wrap;" vertex="1" parent="orders"><mxGeometry x="10" y="230" width="300" height="20" as="geometry"/></mxCell>
        <mxCell id="orders_created" value="    created_at        TIMESTAMP" style="shape=partialRectangle;top=0;left=0;right=0;bottom=0;align=left;verticalAlign=top;spacingLeft=6;spacingRight=6;overflow=hidden;whiteSpace=wrap;" vertex="1" parent="orders"><mxGeometry x="10" y="250" width="300" height="20" as="geometry"/></mxCell>
        <mxCell id="transactions" value="transactions" style="shape=table;startSize=30;container=1;collapsible=0;childLayout=tableLayout;fixedRows=1;rowLines=0;fontStyle=1;align=center;resizeLast=1;" vertex="1" parent="1"><mxGeometry x="440" y="360" width="310" height="250" as="geometry"/></mxCell>
        <mxCell id="tx_pk" value="PK  id                UUID" style="shape=partialRectangle;top=0;left=0;right=0;bottom=0;align=left;verticalAlign=top;spacingLeft=6;spacingRight=6;overflow=hidden;whiteSpace=wrap;" vertex="1" parent="transactions"><mxGeometry x="10" y="30" width="290" height="20" as="geometry"/></mxCell>
        <mxCell id="tx_fk_acc" value="FK  account_id        UUID" style="shape=partialRectangle;top=0;left=0;right=0;bottom=0;align=left;verticalAlign=top;spacingLeft=6;spacingRight=6;overflow=hidden;whiteSpace=wrap;" vertex="1" parent="transactions"><mxGeometry x="10" y="50" width="290" height="20" as="geometry"/></mxCell>
        <mxCell id="tx_fk_order" value="FK  order_id          UUID (NULL)" style="shape=partialRectangle;top=0;left=0;right=0;bottom=0;align=left;verticalAlign=top;spacingLeft=6;spacingRight=6;overflow=hidden;whiteSpace=wrap;" vertex="1" parent="transactions"><mxGeometry x="10" y="70" width="290" height="20" as="geometry"/></mxCell>
        <mxCell id="tx_type" value="    type              VARCHAR(50)" style="shape=partialRectangle;top=0;left=0;right=0;bottom=0;align=left;verticalAlign=top;spacingLeft=6;spacingRight=6;overflow=hidden;whiteSpace=wrap;" vertex="1" parent="transactions"><mxGeometry x="10" y="90" width="290" height="20" as="geometry"/></mxCell>
        <mxCell id="tx_amount" value="    amount            DECIMAL(18,2)" style="shape=partialRectangle;top=0;left=0;right=0;bottom=0;align=left;verticalAlign=top;spacingLeft=6;spacingRight=6;overflow=hidden;whiteSpace=wrap;" vertex="1" parent="transactions"><mxGeometry x="10" y="110" width="290" height="20" as="geometry"/></mxCell>
        <mxCell id="tx_curr" value="    currency          VARCHAR(3)" style="shape=partialRectangle;top=0;left=0;right=0;bottom=0;align=left;verticalAlign=top;spacingLeft=6;spacingRight=6;overflow=hidden;whiteSpace=wrap;" vertex="1" parent="transactions"><mxGeometry x="10" y="130" width="290" height="20" as="geometry"/></mxCell>
        <mxCell id="tx_status" value="    status            VARCHAR(50)" style="shape=partialRectangle;top=0;left=0;right=0;bottom=0;align=left;verticalAlign=top;spacingLeft=6;spacingRight=6;overflow=hidden;whiteSpace=wrap;" vertex="1" parent="transactions"><mxGeometry x="10" y="150" width="290" height="20" as="geometry"/></mxCell>
        <mxCell id="tx_created" value="    created_at        TIMESTAMP" style="shape=partialRectangle;top=0;left=0;right=0;bottom=0;align=left;verticalAlign=top;spacingLeft=6;spacingRight=6;overflow=hidden;whiteSpace=wrap;" vertex="1" parent="transactions"><mxGeometry x="10" y="190" width="290" height="20" as="geometry"/></mxCell>
        <mxCell id="portfolios" value="portfolios" style="shape=table;startSize=30;container=1;collapsible=0;childLayout=tableLayout;fixedRows=1;rowLines=0;fontStyle=1;align=center;resizeLast=1;" vertex="1" parent="1"><mxGeometry x="840" y="360" width="300" height="230" as="geometry"/></mxCell>
        <mxCell id="port_pk" value="PK  id                UUID" style="shape=partialRectangle;top=0;left=0;right=0;bottom=0;align=left;verticalAlign=top;spacingLeft=6;spacingRight=6;overflow=hidden;whiteSpace=wrap;" vertex="1" parent="portfolios"><mxGeometry x="10" y="30" width="280" height="20" as="geometry"/></mxCell>
        <mxCell id="port_fk_acc" value="FK  account_id        UUID" style="shape=partialRectangle;top=0;left=0;right=0;bottom=0;align=left;verticalAlign=top;spacingLeft=6;spacingRight=6;overflow=hidden;whiteSpace=wrap;" vertex="1" parent="portfolios"><mxGeometry x="10" y="50" width="280" height="20" as="geometry"/></mxCell>
        <mxCell id="port_fk_inst" value="FK  instrument_id     UUID" style="shape=partialRectangle;top=0;left=0;right=0;bottom=0;align=left;verticalAlign=top;spacingLeft=6;spacingRight=6;overflow=hidden;whiteSpace=wrap;" vertex="1" parent="portfolios"><mxGeometry x="10" y="70" width="280" height="20" as="geometry"/></mxCell>
        <mxCell id="port_qty" value="    quantity          DECIMAL(18,4)" style="shape=partialRectangle;top=0;left=0;right=0;bottom=0;align=left;verticalAlign=top;spacingLeft=6;spacingRight=6;overflow=hidden;whiteSpace=wrap;" vertex="1" parent="portfolios"><mxGeometry x="10" y="90" width="280" height="20" as="geometry"/></mxCell>
        <mxCell id="port_avg" value="    avg_price         DECIMAL(18,2)" style="shape=partialRectangle;top=0;left=0;right=0;bottom=0;align=left;verticalAlign=top;spacingLeft=6;spacingRight=6;overflow=hidden;whiteSpace=wrap;" vertex="1" parent="portfolios"><mxGeometry x="10" y="110" width="280" height="20" as="geometry"/></mxCell>
        <mxCell id="port_curr" value="    current_price     DECIMAL(18,2)" style="shape=partialRectangle;top=0;left=0;right=0;bottom=0;align=left;verticalAlign=top;spacingLeft=6;spacingRight=6;overflow=hidden;whiteSpace=wrap;" vertex="1" parent="portfolios"><mxGeometry x="10" y="130" width="280" height="20" as="geometry"/></mxCell>
        <mxCell id="port_upd" value="    updated_at        TIMESTAMP" style="shape=partialRectangle;top=0;left=0;right=0;bottom=0;align=left;verticalAlign=top;spacingLeft=6;spacingRight=6;overflow=hidden;whiteSpace=wrap;" vertex="1" parent="portfolios"><mxGeometry x="10" y="150" width="280" height="20" as="geometry"/></mxCell>
        <mxCell id="commission_rules" value="commission_rules" style="shape=table;startSize=30;container=1;collapsible=0;childLayout=tableLayout;fixedRows=1;rowLines=0;fontStyle=1;align=center;resizeLast=1;" vertex="1" parent="1"><mxGeometry x="1200" y="40" width="300" height="270" as="geometry"/></mxCell>
        <mxCell id="comm_pk" value="PK  id                UUID" style="shape=partialRectangle;top=0;left=0;right=0;bottom=0;align=left;verticalAlign=top;spacingLeft=6;spacingRight=6;overflow=hidden;whiteSpace=wrap;" vertex="1" parent="commission_rules"><mxGeometry x="10" y="30" width="280" height="20" as="geometry"/></mxCell>
        <mxCell id="comm_type_inst" value="    instrument_type   VARCHAR(50)" style="shape=partialRectangle;top=0;left=0;right=0;bottom=0;align=left;verticalAlign=top;spacingLeft=6;spacingRight=6;overflow=hidden;whiteSpace=wrap;" vertex="1" parent="commission_rules"><mxGeometry x="10" y="50" width="280" height="20" as="geometry"/></mxCell>
        <mxCell id="comm_type" value="    commission_type   VARCHAR(50)" style="shape=partialRectangle;top=0;left=0;right=0;bottom=0;align=left;verticalAlign=top;spacingLeft=6;spacingRight=6;overflow=hidden;whiteSpace=wrap;" vertex="1" parent="commission_rules"><mxGeometry x="10" y="70" width="280" height="20" as="geometry"/></mxCell>
        <mxCell id="comm_val" value="    value             DECIMAL(10,4)" style="shape=partialRectangle;top=0;left=0;right=0;bottom=0;align=left;verticalAlign=top;spacingLeft=6;spacingRight=6;overflow=hidden;whiteSpace=wrap;" vertex="1" parent="commission_rules"><mxGeometry x="10" y="90" width="280" height="20" as="geometry"/></mxCell>
        <mxCell id="comm_min" value="    min_commission    DECIMAL(18,2)" style="shape=partialRectangle;top=0;left=0;right=0;bottom=0;align=left;verticalAlign=top;spacingLeft=6;spacingRight=6;overflow=hidden;whiteSpace=wrap;" vertex="1" parent="commission_rules"><mxGeometry x="10" y="130" width="280" height="20" as="geometry"/></mxCell>
        <mxCell id="comm_max" value="    max_commission    DECIMAL(18,2)" style="shape=partialRectangle;top=0;left=0;right=0;bottom=0;align=left;verticalAlign=top;spacingLeft=6;spacingRight=6;overflow=hidden;whiteSpace=wrap;" vertex="1" parent="commission_rules"><mxGeometry x="10" y="150" width="280" height="20" as="geometry"/></mxCell>
        <mxCell id="comm_valid_from" value="    valid_from        TIMESTAMP" style="shape=partialRectangle;top=0;left=0;right=0;bottom=0;align=left;verticalAlign=top;spacingLeft=6;spacingRight=6;overflow=hidden;whiteSpace=wrap;" vertex="1" parent="commission_rules"><mxGeometry x="10" y="170" width="280" height="20" as="geometry"/></mxCell>
        <mxCell id="comm_valid_to" value="    valid_to          TIMESTAMP" style="shape=partialRectangle;top=0;left=0;right=0;bottom=0;align=left;verticalAlign=top;spacingLeft=6;spacingRight=6;overflow=hidden;whiteSpace=wrap;" vertex="1" parent="commission_rules"><mxGeometry x="10" y="190" width="280" height="20" as="geometry"/></mxCell>
        <mxCell id="withdrawal_requests" value="withdrawal_requests" style="shape=table;startSize=30;container=1;collapsible=0;childLayout=tableLayout;fixedRows=1;rowLines=0;fontStyle=1;align=center;resizeLast=1;" vertex="1" parent="1"><mxGeometry x="1200" y="380" width="310" height="270" as="geometry"/></mxCell>
        <mxCell id="wd_pk" value="PK  id                UUID" style="shape=partialRectangle;top=0;left=0;right=0;bottom=0;align=left;verticalAlign=top;spacingLeft=6;spacingRight=6;overflow=hidden;whiteSpace=wrap;" vertex="1" parent="withdrawal_requests"><mxGeometry x="10" y="30" width="290" height="20" as="geometry"/></mxCell>
        <mxCell id="wd_fk_acc" value="FK  account_id        UUID" style="shape=partialRectangle;top=0;left=0;right=0;bottom=0;align=left;verticalAlign=top;spacingLeft=6;spacingRight=6;overflow=hidden;whiteSpace=wrap;" vertex="1" parent="withdrawal_requests"><mxGeometry x="10" y="50" width="290" height="20" as="geometry"/></mxCell>
        <mxCell id="wd_amount" value="    amount            DECIMAL(18,2)" style="shape=partialRectangle;top=0;left=0;right=0;bottom=0;align=left;verticalAlign=top;spacingLeft=6;spacingRight=6;overflow=hidden;whiteSpace=wrap;" vertex="1" parent="withdrawal_requests"><mxGeometry x="10" y="70" width="290" height="20" as="geometry"/></mxCell>
        <mxCell id="wd_curr" value="    currency          VARCHAR(3)" style="shape=partialRectangle;top=0;left=0;right=0;bottom=0;align=left;verticalAlign=top;spacingLeft=6;spacingRight=6;overflow=hidden;whiteSpace=wrap;" vertex="1" parent="withdrawal_requests"><mxGeometry x="10" y="90" width="290" height="20" as="geometry"/></mxCell>
        <mxCell id="wd_status" value="    status            VARCHAR(50)" style="shape=partialRectangle;top=0;left=0;right=0;bottom=0;align=left;verticalAlign=top;spacingLeft=6;spacingRight=6;overflow=hidden;whiteSpace=wrap;" vertex="1" parent="withdrawal_requests"><mxGeometry x="10" y="130" width="290" height="20" as="geometry"/></mxCell>
        <mxCell id="wd_compl" value="    compliance_status VARCHAR(50)" style="shape=partialRectangle;top=0;left=0;right=0;bottom=0;align=left;verticalAlign=top;spacingLeft=6;spacingRight=6;overflow=hidden;whiteSpace=wrap;" vertex="1" parent="withdrawal_requests"><mxGeometry x="10" y="150" width="290" height="20" as="geometry"/></mxCell>
        <mxCell id="wd_created" value="    created_at        TIMESTAMP" style="shape=partialRectangle;top=0;left=0;right=0;bottom=0;align=left;verticalAlign=top;spacingLeft=6;spacingRight=6;overflow=hidden;whiteSpace=wrap;" vertex="1" parent="withdrawal_requests"><mxGeometry x="10" y="190" width="290" height="20" as="geometry"/></mxCell>
        <mxCell id="notifications" value="notifications" style="shape=table;startSize=30;container=1;collapsible=0;childLayout=tableLayout;fixedRows=1;rowLines=0;fontStyle=1;align=center;resizeLast=1;" vertex="1" parent="1"><mxGeometry x="40" y="760" width="310" height="250" as="geometry"/></mxCell>
        <mxCell id="notif_pk" value="PK  id                UUID" style="shape=partialRectangle;top=0;left=0;right=0;bottom=0;align=left;verticalAlign=top;spacingLeft=6;spacingRight=6;overflow=hidden;whiteSpace=wrap;" vertex="1" parent="notifications"><mxGeometry x="10" y="30" width="290" height="20" as="geometry"/></mxCell>
        <mxCell id="notif_fk_user" value="FK  user_id           UUID" style="shape=partialRectangle;top=0;left=0;right=0;bottom=0;align=left;verticalAlign=top;spacingLeft=6;spacingRight=6;overflow=hidden;whiteSpace=wrap;" vertex="1" parent="notifications"><mxGeometry x="10" y="50" width="290" height="20" as="geometry"/></mxCell>
        <mxCell id="notif_type" value="    type              VARCHAR(50)" style="shape=partialRectangle;top=0;left=0;right=0;bottom=0;align=left;verticalAlign=top;spacingLeft=6;spacingRight=6;overflow=hidden;whiteSpace=wrap;" vertex="1" parent="notifications"><mxGeometry x="10" y="70" width="290" height="20" as="geometry"/></mxCell>
        <mxCell id="notif_channel" value="    channel           VARCHAR(50)" style="shape=partialRectangle;top=0;left=0;right=0;bottom=0;align=left;verticalAlign=top;spacingLeft=6;spacingRight=6;overflow=hidden;whiteSpace=wrap;" vertex="1" parent="notifications"><mxGeometry x="10" y="90" width="290" height="20" as="geometry"/></mxCell>
        <mxCell id="notif_status" value="    status            VARCHAR(50)" style="shape=partialRectangle;top=0;left=0;right=0;bottom=0;align=left;verticalAlign=top;spacingLeft=6;spacingRight=6;overflow=hidden;whiteSpace=wrap;" vertex="1" parent="notifications"><mxGeometry x="10" y="150" width="290" height="20" as="geometry"/></mxCell>
        <mxCell id="notif_created" value="    created_at        TIMESTAMP" style="shape=partialRectangle;top=0;left=0;right=0;bottom=0;align=left;verticalAlign=top;spacingLeft=6;spacingRight=6;overflow=hidden;whiteSpace=wrap;" vertex="1" parent="notifications"><mxGeometry x="10" y="170" width="290" height="20" as="geometry"/></mxCell>
        <mxCell id="rel_users_accounts" value="" style="endArrow=ERmany;startArrow=ERone;edgeStyle=orthogonalEdgeStyle;html=1;startFill=0;endFill=0;" edge="1" parent="1" source="users" target="accounts"><mxGeometry relative="1" as="geometry"/></mxCell>
        <mxCell id="rel_users_accounts_label1" value="1" style="text;html=1;strokeColor=none;fillColor=none;align=center;" vertex="1" parent="1"><mxGeometry x="330" y="155" width="20" height="20" as="geometry"/></mxCell>
        <mxCell id="rel_users_accounts_labelN" value="N" style="text;html=1;strokeColor=none;fillColor=none;align=center;" vertex="1" parent="1"><mxGeometry x="385" y="155" width="20" height="20" as="geometry"/></mxCell>
        <mxCell id="rel_accounts_orders" value="" style="endArrow=ERmany;startArrow=ERone;edgeStyle=orthogonalEdgeStyle;html=1;startFill=0;endFill=0;" edge="1" parent="1" source="accounts" target="orders"><mxGeometry relative="1" as="geometry"/></mxCell>
        <mxCell id="rel_accounts_orders_label1" value="1" style="text;html=1;strokeColor=none;fillColor=none;align=center;" vertex="1" parent="1"><mxGeometry x="520" y="330" width="20" height="20" as="geometry"/></mxCell>
        <mxCell id="rel_accounts_orders_labelN" value="N" style="text;html=1;strokeColor=none;fillColor=none;align=center;" vertex="1" parent="1"><mxGeometry x="185" y="330" width="20" height="20" as="geometry"/></mxCell>
        <mxCell id="rel_instr_orders" value="" style="endArrow=ERmany;startArrow=ERone;edgeStyle=orthogonalEdgeStyle;html=1;startFill=0;endFill=0;" edge="1" parent="1" source="instruments" target="orders"><mxGeometry relative="1" as="geometry"/></mxCell>
        <mxCell id="rel_accounts_trans" value="" style="endArrow=ERmany;startArrow=ERone;edgeStyle=orthogonalEdgeStyle;html=1;startFill=0;endFill=0;" edge="1" parent="1" source="accounts" target="transactions"><mxGeometry relative="1" as="geometry"/></mxCell>
        <mxCell id="rel_orders_trans" value="" style="endArrow=ERzeroToOne;startArrow=ERone;edgeStyle=orthogonalEdgeStyle;html=1;startFill=0;endFill=0;dashed=1;" edge="1" parent="1" source="orders" target="transactions"><mxGeometry relative="1" as="geometry"/></mxCell>
        <mxCell id="rel_accounts_port" value="" style="endArrow=ERmany;startArrow=ERone;edgeStyle=orthogonalEdgeStyle;html=1;startFill=0;endFill=0;" edge="1" parent="1" source="accounts" target="portfolios"><mxGeometry relative="1" as="geometry"/></mxCell>
        <mxCell id="rel_instr_port" value="" style="endArrow=ERmany;startArrow=ERone;edgeStyle=orthogonalEdgeStyle;html=1;startFill=0;endFill=0;" edge="1" parent="1" source="instruments" target="portfolios"><mxGeometry relative="1" as="geometry"/></mxCell>
        <mxCell id="rel_accounts_wd" value="" style="endArrow=ERmany;startArrow=ERone;edgeStyle=orthogonalEdgeStyle;html=1;startFill=0;endFill=0;" edge="1" parent="1" source="accounts" target="withdrawal_requests"><mxGeometry relative="1" as="geometry"/></mxCell>
        <mxCell id="rel_users_notif" value="" style="endArrow=ERmany;startArrow=ERone;edgeStyle=orthogonalEdgeStyle;html=1;startFill=0;endFill=0;" edge="1" parent="1" source="users" target="notifications"><mxGeometry relative="1" as="geometry"/></mxCell>
        <mxCell id="rel_users_notif_label1" value="1" style="text;html=1;strokeColor=none;fillColor=none;align=center;" vertex="1" parent="1"><mxGeometry x="155" y="500" width="20" height="20" as="geometry"/></mxCell>
        <mxCell id="rel_users_notif_labelN" value="N" style="text;html=1;strokeColor=none;fillColor=none;align=center;" vertex="1" parent="1"><mxGeometry x="155" y="740" width="20" height="20" as="geometry"/></mxCell>
      </root>
    </mxGraphModel>
  </diagram>
</mxfile>
```

---

> **Инструкция по импорту в draw.io:**
> 1. Открой [draw.io](https://app.diagrams.net)
> 2. Меню: **Extras → Edit Diagram**
> 3. Выдели весь текст в поле и удали его
> 4. Вставь XML из блока выше
> 5. Нажми **OK**
> 6. Диаграмма появится на холсте
> 7. Сохрани: **File → Save As** → выбери формат `.xml` или `.drawio`
