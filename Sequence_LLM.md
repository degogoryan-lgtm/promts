Шаг 1 — Описание трёх сценариев текстом
Сценарий 1 — Подача торговой заявки
Участники:

Инвестор (actor)

Экран заявок (boundary)

Сервис заявок (control)

Сервис риска (control)

Сервис комиссий (control)

Биржевой шлюз (control)

Сервис уведомлений (control)

База данных (entity)

Пошаговое описание:

Инвестор открывает экран заявок → Экран заявок запрашивает у Сервиса заявок доступные инструменты и текущий баланс.

Инвестор заполняет форму: инструмент, направление (покупка/продажа), количество, тип заявки (рыночная/лимитная), для лимитной — цену. Нажимает «Отправить».

Экран заявок передаёт данные в Сервис заявок.

Сервис заявок вычисляет сумму сделки (количество × цена или рыночная цена). Проверяет лимит:

alt [сумма > 10 000 000 рублей] → Сервис заявок возвращает ошибку «Превышен лимит», Экран показывает сообщение, процесс завершается.

else [сумма ≤ 10 000 000] → продолжение.

Сервис заявок параллельно вызывает:

Сервис риска: проверить, не превышен ли дневной лимит по инструменту, и достаточно ли средств на счёте.

Сервис комиссий: рассчитать комиссию (в зависимости от типа инструмента и суммы).

Если одна из проверок не пройдена → Сервис заявок возвращает отказ (статус FAILED) с причиной. Экран показывает ошибку.

Если обе проверки успешны → Сервис заявок сохраняет заявку в БД со статусом PROCESSING, блокирует сумму (вместе со средствами на комиссию).

Сервис заявок отправляет заявку в Биржевой шлюз.

Биржевой шлюз передаёт заявку на биржу и ждёт ответа.

alt [биржа подтвердила исполнение] → Биржевой шлюз возвращает статус COMPLETED, Сервис заявок обновляет заявку в БД, списывает средства, фиксирует комиссию.

else [биржа отклонила] → возвращает FAILED, Сервис заявок разблокирует средства, статус FAILED.

Сервис заявок асинхронно вызывает Сервис уведомлений → отправляет инвестору email/push о результате.

Сценарий 2 — Вывод средств
Участники:

Инвестор (actor)

Экран вывода средств (boundary)

Сервис баланса (control)

Сервис комплаенса (control)

Платёжный шлюз (control)

Сервис уведомлений (control)

База данных (entity)

Пошаговое описание:

Инвестор открывает экран вывода средств → Экран запрашивает у Сервиса баланса доступные для вывода суммы по счетам.

Инвестор вводит сумму, выбирает счёт и реквизиты, нажимает «Вывести».

Экран передаёт запрос в Сервис баланса.

Сервис баланса проверяет: доступный остаток ≥ запрашиваемая сумма.

alt [недостаточно средств] → отказ с сообщением «Недостаточно средств», Экран показывает ошибку.

else [достаточно] → продолжение.

Сервис баланса вызывает Сервис комплаенса для проверки KYC/AML по данному запросу (сумма, частота, история).

alt [комплаенс не пройден] → отказ с причиной (COMPLIANCE_REJECTED), процесс завершён.

else [пройден] → продолжение.

Сервис баланса блокирует запрашиваемую сумму на счёте (помечает как «в процессе вывода»).

Сервис баланса вызывает Платёжный шлюз с поручением перевести деньги.

Платёжный шлюз обрабатывает перевод.

alt [платёж успешен] → Платёжный шлюз подтверждает, Сервис баланса списывает сумму окончательно, статус COMPLETED.

else [платёж отклонён банком или техническая ошибка] → Платёжный шлюз возвращает ошибку, Сервис баланса разблокирует сумму, статус FAILED.

Сервис баланса записывает результат в БД (заявка на вывод).

Сервис баланса асинхронно уведомляет Сервис уведомлений → отправляет инвестору статус вывода.

Сценарий 3 — Просмотр портфеля
Участники:

Инвестор (actor)

Экран портфеля (boundary)

Сервис портфеля (control)

Сервис рыночных данных (control)

Кэш Redis (entity)

База данных (entity)

Пошаговое описание:

Инвестор открывает экран портфеля.

Экран портфеля обращается к Сервису портфеля с запросом на получение данных.

Сервис портфеля проверяет наличие данных в кэше Redis (ключ: portfolio:{user_id}).

alt [данные есть в кэше и не устарели (TTL 60 сек)] → Сервис портфеля возвращает данные из Redis на экран.

else [кэш отсутствует или устарел] → Сервис портфеля выполняет параллельные запросы:

к Сервису рыночных данных — получить текущие котировки по всем инструментам в портфеле;

к Базе данных — получить позиции пользователя (инструмент, количество, средняя цена).

Сервис портфеля агрегирует данные: вычисляет текущую стоимость позиций, доходность, валютный пересчёт в рубли.

Сервис портфеля сохраняет агрегированные данные в Redis с TTL 60 секунд.

Сервис портфеля возвращает результат на Экран портфеля.

Экран отображает портфель.

Если инвестор нажимает кнопку «Обновить» вручную → ref (ссылка на сценарий принудительного обновления, который повторяет шаги 3-6, игнорируя кэш). Шаг 2 — Три блока кода для sequencediagram.org
Ниже представлены три диаграммы в синтаксисе, совместимом с сайтом sequencediagram.org. Скопируйте каждый блок по отдельности, вставьте на сайт и нажмите Refresh.

// Сценарий 1 — Подача торговой заявки
text
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
Investor -> OrderScreen: Заполнить форму (инструмент, направление, кол-во, тип, цена) и нажать "Отправить"
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

    alt Проверка риска не пройдена или недостаточно средств
        OrderService --> OrderScreen: Отказ (FAILED) с причиной
        OrderScreen --> Investor: Сообщение об ошибке
    else Проверки успешны
        OrderService -> DB: saveOrder(status=PROCESSING, блокировка суммы)
        DB --> OrderService: ok
        OrderService -> ExchangeGateway: submitOrder(orderData)
        ExchangeGateway --> ExchangeGateway: Передача на биржу

        alt Биржа подтвердила
            ExchangeGateway --> OrderService: status=COMPLETED
            OrderService -> DB: updateOrder(COMPLETED, списание средств)
            DB --> OrderService: ok
        else Биржа отклонила
            ExchangeGateway --> OrderService: status=FAILED
            OrderService -> DB: updateOrder(FAILED, разблокировка средств)
            DB --> OrderService: ok
        end

        OrderService -> NotifyService: sendNotification(userId, result)
        NotifyService --> OrderService: notificationSent
        OrderService --> OrderScreen: finalStatus
        OrderScreen --> Investor: Отображение статуса заявки
    end
end
// Сценарий 2 — Вывод средств
text
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
Investor -> WithdrawScreen: Ввести сумму, реквизиты, нажать "Вывести"
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
        BalanceService -> DB: Блокировка суммы (pending withdrawal)
        DB --> BalanceService: blocked
        BalanceService -> PaymentGateway: transferMoney(amount, requisites)
        
        alt Платеж успешен
            PaymentGateway --> BalanceService: success
            BalanceService -> DB: Списание суммы, статус COMPLETED
            DB --> BalanceService: ok
        else Платеж отклонен / ошибка
            PaymentGateway --> BalanceService: failure
            BalanceService -> DB: Разблокировка суммы, статус FAILED
            DB --> BalanceService: ok
        end
        
        BalanceService -> NotifyService: sendWithdrawalStatus(userId, status)
        NotifyService --> BalanceService: sent
        BalanceService --> WithdrawScreen: withdrawalResult
        WithdrawScreen --> Investor: Отображение статуса вывода
    end
end
// Сценарий 3 — Просмотр портфеля
text
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
    PortfolioService -> Redis: (нет данных)
    Redis --> PortfolioService: null
    
    par Запрос рыночных данных
        PortfolioService -> MarketDataService: getQuotes(instrumentIds)
        MarketDataService --> PortfolioService: quotes
    and Запрос позиций из БД
        PortfolioService -> DB: getUserPositions(userId)
        DB --> PortfolioService: positions
    end
    
    PortfolioService -> PortfolioService: Агрегация (стоимость, доходность, пересчет валют)
    PortfolioService -> Redis: setex(portfolio:userId, 60, aggregatedData)
    Redis --> PortfolioService: ok
    PortfolioService --> PortfolioScreen: aggregatedData
    PortfolioScreen --> Investor: Отобразить портфель
end

Investor -> PortfolioScreen: Нажать "Обновить" вручную
ref over Investor,PortfolioScreen,PortfolioService,MarketDataService,Redis,DB: Принудительное обновление (игнорирование кэша)
PortfolioService --> PortfolioScreen: refreshedData
PortfolioScreen --> Investor: Обновленный портфель

