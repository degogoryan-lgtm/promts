# Sequence-диаграммы: Инвестиционная платформа (ЛК Брокера)

---

## Шаг 1 — Описание сценариев текстом

### Сценарий 1 — Подача торговой заявки

**Участники:** Инвестор, Экран заявок, Сервис заявок,
Сервис риска, Сервис комиссий, Биржевой шлюз,
Сервис уведомлений, База данных.

**Пошаговое описание:**

1. Инвестор открывает экран заявок → Экран запрашивает
   у Сервиса заявок доступные инструменты и баланс.

2. Инвестор заполняет форму и нажимает «Отправить» →
   Экран передаёт данные в Сервис заявок.

3. Сервис заявок проверяет сумму сделки:
   - **alt** [сумма > 10 000 000 руб.] → ошибка
     «Превышен лимит», процесс завершается.
   - **else** [сумма ≤ 10 000 000] → продолжение.

4. Сервис заявок параллельно вызывает:
   - Сервис риска — проверить достаточность средств.
   - Сервис комиссий — рассчитать комиссию по типу
     инструмента (FIXED / PERCENTAGE / MIXED).

5. Если проверки не пройдены → отказ со статусом FAILED.
   Если пройдены → заявка сохраняется в БД
   со статусом PROCESSING, сумма блокируется.

6. Сервис заявок отправляет заявку в Биржевой шлюз:
   - **alt** [биржа подтвердила] → статус COMPLETED,
     средства списываются.
   - **else** [биржа отклонила] → статус FAILED,
     средства разблокируются.

7. Сервис заявок асинхронно вызывает Сервис уведомлений
   → инвестор получает email/push о результате.

---

### Сценарий 2 — Вывод средств

**Участники:** Инвестор, Экран вывода средств,
Сервис баланса, Сервис комплаенса, Платёжный шлюз,
Сервис уведомлений, База данных.

**Пошаговое описание:**

1. Инвестор открывает экран вывода → Экран запрашивает
   у Сервиса баланса доступные суммы по счетам.

2. Инвестор вводит сумму и реквизиты, нажимает «Вывести».

3. Сервис баланса проверяет остаток:
   - **alt** [недостаточно средств] → ошибка,
     процесс завершается.
   - **else** [достаточно] → продолжение.

4. Сервис баланса вызывает Сервис комплаенса
   для проверки KYC/AML:
   - **alt** [не пройдена] → COMPLIANCE_REJECTED,
     процесс завершается.
   - **else** [пройдена] → продолжение.

5. Сервис баланса блокирует сумму на счёте.

6. Сервис баланса вызывает Платёжный шлюз:
   - **alt** [платёж успешен] → сумма списывается,
     статус COMPLETED.
   - **else** [платёж отклонён] → сумма разблокируется,
     статус FAILED.

7. Результат записывается в БД.
   Сервис баланса асинхронно уведомляет инвестора.

---

### Сценарий 3 — Просмотр портфеля

**Участники:** Инвестор, Экран портфеля,
Сервис портфеля, Сервис рыночных данных,
Кэш Redis, База данных.

**Пошаговое описание:**

1. Инвестор открывает экран портфеля →
   Экран обращается к Сервису портфеля.

2. Сервис портфеля проверяет кэш Redis
   (ключ: portfolio:{user_id}, TTL 60 сек):
   - **alt** [данные есть в кэше] → быстрый ответ
     из Redis на экран.
   - **else** [кэш пустой или устарел] → параллельные
     запросы к Сервису рыночных данных (котировки)
     и к БД (позиции пользователя).

3. Сервис портфеля агрегирует данные:
   текущая стоимость, доходность, пересчёт валют в рубли.

4. Результат сохраняется в Redis с TTL 60 секунд.

5. Данные возвращаются на экран.

6. Если инвестор нажимает «Обновить» вручную →
   ref на принудительное обновление
   (повтор шагов 2-5, кэш игнорируется).

---

## Шаг 2 — Код для sequencediagram.org

Скопируй каждый блок на сайт и нажми Refresh.

---

### Сценарий 1 — Подача торговой заявки

```
actor "Инвестор" as Investor
boundary "Экран заявок" as OrderScreen
control "Сервис заявок" as OrderService
control "Сервис риска" as RiskService
control "Сервис комиссий" as CommissionService
control "Биржевой шлюз" as ExchangeGateway
control "Сервис уведомлений" as NotifyService
entity "База данных" as DB

Investor -> OrderScreen: Открыть экран заявок
OrderScreen -> OrderService: Запросить инструменты и баланс
OrderService --> OrderScreen: Список инструментов, баланс
Investor -> OrderScreen: Заполнить форму и нажать "Отправить"
OrderScreen -> OrderService: orderRequest(data)

alt Сумма > 10 000 000 руб.
    OrderService --> OrderScreen: Ошибка "Превышен лимит"
    OrderScreen --> Investor: Сообщение об ошибке
else Сумма <= 10 000 000 руб.
    par Проверка риска
        OrderService -> RiskService: checkRisk(orderData)
        RiskService --> OrderService: riskResult
    and Расчёт комиссии
        OrderService -> CommissionService: calculateCommission(orderData)
        CommissionService --> OrderService: commissionAmount
    end

    alt Проверка риска не пройдена
        OrderService --> OrderScreen: Отказ (FAILED) с причиной
        OrderScreen --> Investor: Сообщение об ошибке
    else Проверки успешны
        OrderService -> DB: saveOrder(status=PROCESSING)
        DB --> OrderService: ok
        OrderService -> ExchangeGateway: submitOrder(orderData)

        alt Биржа подтвердила
            ExchangeGateway --> OrderService: status=COMPLETED
            OrderService -> DB: updateOrder(COMPLETED)
            DB --> OrderService: ok
        else Биржа отклонила
            ExchangeGateway --> OrderService: status=FAILED
            OrderService -> DB: updateOrder(FAILED)
            DB --> OrderService: ok
        end

        OrderService ->> NotifyService: sendNotification(userId, result)
        OrderService --> OrderScreen: finalStatus
        OrderScreen --> Investor: Отображение статуса заявки
    end
end
```

---

### Сценарий 2 — Вывод средств

```
actor "Инвестор" as Investor
boundary "Экран вывода средств" as WithdrawScreen
control "Сервис баланса" as BalanceService
control "Сервис комплаенса" as ComplianceService
control "Платежный шлюз" as PaymentGateway
control "Сервис уведомлений" as NotifyService
entity "База данных" as DB

Investor -> WithdrawScreen: Открыть экран вывода
WithdrawScreen -> BalanceService: Запросить доступный баланс
BalanceService --> WithdrawScreen: Баланс по счетам
Investor -> WithdrawScreen: Ввести сумму и реквизиты
WithdrawScreen -> BalanceService: withdrawalRequest(amount, account)

alt Недостаточно средств
    BalanceService --> WithdrawScreen: Ошибка "Недостаточно средств"
    WithdrawScreen --> Investor: Сообщение об ошибке
else Достаточно средств
    BalanceService -> ComplianceService: checkKYCAML(userId, amount)
    alt Комплаенс не пройден
        ComplianceService --> BalanceService: COMPLIANCE_REJECTED
        BalanceService --> WithdrawScreen: Отказ с причиной
        WithdrawScreen --> Investor: Сообщение об ошибке
    else Комплаенс пройден
        ComplianceService --> BalanceService: OK
        BalanceService -> DB: Блокировка суммы
        DB --> BalanceService: blocked
        BalanceService -> PaymentGateway: transferMoney(amount, requisites)

        alt Платеж успешен
            PaymentGateway --> BalanceService: success
            BalanceService -> DB: Списание, статус COMPLETED
            DB --> BalanceService: ok
        else Платеж отклонен
            PaymentGateway --> BalanceService: failure
            BalanceService -> DB: Разблокировка, статус FAILED
            DB --> BalanceService: ok
        end

        BalanceService ->> NotifyService: sendWithdrawalStatus(userId, status)
        BalanceService --> WithdrawScreen: withdrawalResult
        WithdrawScreen --> Investor: Отображение статуса вывода
    end
end
```

---

### Сценарий 3 — Просмотр портфеля

```
actor "Инвестор" as Investor
boundary "Экран портфеля" as PortfolioScreen
control "Сервис портфеля" as PortfolioService
control "Сервис рыночных данных" as MarketDataService
entity "Кэш Redis" as Redis
entity "База данных" as DB

Investor -> PortfolioScreen: Открыть экран портфеля
PortfolioScreen -> PortfolioService: getPortfolio(userId)
PortfolioService -> Redis: get(portfolio:userId)

alt Данные есть в кэше (TTL 60 сек)
    Redis --> PortfolioService: cachedData
    PortfolioService --> PortfolioScreen: portfolioData
    PortfolioScreen --> Investor: Отобразить портфель
else Кэш отсутствует / устарел
    Redis --> PortfolioService: null

    par Запрос рыночных данных
        PortfolioService -> MarketDataService: getQuotes(instrumentIds)
        MarketDataService --> PortfolioService: quotes
    and Запрос позиций из БД
        PortfolioService -> DB: getUserPositions(userId)
        DB --> PortfolioService: positions
    end

    PortfolioService -> PortfolioService: Агрегация (стоимость, доходность, валюты)
    PortfolioService -> Redis: setex(portfolio:userId, 60, aggregatedData)
    Redis --> PortfolioService: ok
    PortfolioService --> PortfolioScreen: aggregatedData
    PortfolioScreen --> Investor: Отобразить портфель
end

Investor -> PortfolioScreen: Нажать "Обновить" вручную
ref over Investor,PortfolioScreen,PortfolioService,MarketDataService,Redis,DB: Принудительное обновление (игнорирование кэша)
PortfolioService --> PortfolioScreen: refreshedData
PortfolioScreen --> Investor: Обновленный портфель
```

