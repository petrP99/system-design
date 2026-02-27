# Сценарии платежной системы

## Успешный платеж ([пример ответа](..\responses\paymentCompleted.json))
1. **Клиент** отправляет синхронный REST запрос через **Kong API Gateway** в **Payments Service**.
2. **Payments Service**:
    * Проверяет данные платежа.
    * Отправляет синхронный gRPC запрос в **Wallet Service** для резервирования средств.
    * Отправляет синхронный gRPC запрос в **Anti-Fraud Service** для проверки риска.
    * Сохраняет платеж в статусе `IN_PROGRESS` в **Payments DB**.
    * Отправляет клиенту "Платеж принят, обрабатывается".
    * Добавляет запись в таблицу **outbox**.
3. **Outbox Worker**:
    * Обнаруживает новую запись в outbox.
    * Публикует событие `payment.created` в **Kafka**.
4. **Wallet Service**:
    * Подписывается на события `payment.created`.
    * Проверяет наличие резервирования.
    * Списывает средства (уменьшает `total_balance`, удаляет `reserved_amount`).
    * Публикует событие `payment.completed` в **Kafka**.
5. **Payments Service**:
    * Получает событие `payments.completed` в Kafka.
    * Обновляет статус платежа на `COMPLETED`.
6. **Core Integration (ACL)**:
    * Подписывается на события `payment.completed`.
    * Преобразует событие в формат Core Banking (`json` -> `xml`).
    * Отправляет синхронный SOAP запрос в **Core Banking**.
7. **Notifications Service**:
    * Подписывается на события `payment.completed`.
    * Формирует уведомление.
    * Доставляет уведомление через SMS/email провайдеры клиенту.
8. **Callbacks Service**:
    * Подписывается на события `payment.completed`.
    * Отправляет Webhook в **Merchant**.

---

## Неуспешный платеж
1. **Клиент** отправляет синхронный REST запрос через **Kong API Gateway** в **Payments Service**.
2. **Payments Service**:
    * Проверяет данные платежа.
    * Отправляет синхронный gRPC запрос в **Wallet Service** для резервирования средств.
    * Отправляет синхронный gRPC запрос в **Anti-Fraud Service** для проверки риска.
    * **Если недостаточно средств или обнаружено мошенничество**:
        * Сохраняет платеж в статусе `FAILED` в **Payments DB**.
        * Публикует событие `payment.failed` в **Kafka**.
3. **Wallet Service**:
    * Читает сообщение из `payment.failed` в Kafka.
    * **Если находит резерв суммы по payment_id**:
        * Увеличивает `available_balance`, Удаляет `reserved_amount`.
        * Не изменяет `total_balance`.
4. **Notifications Service**:
    * Подписывается на события `payment.failed`.
    * Формирует уведомление об ошибке.
    * Доставляет уведомление клиенту об ошибке.
5. **Callbacks Service**:
    * Подписывается на события `payment.failed`.
    * Отправляет Webhook в **Merchant**.

---

## Ключевые отношения

### Кратко:
* **Payments → Wallet**: Customer/Supplier (gRPC)
* **Payments → Anti-Fraud**: Customer/Supplier (gRPC)
* **Payments → Core Banking**: Anti-Corruption Layer (Kafka + REST)
* **Payments → Callbacks**: Published Language (Kafka)
* **Payments → Notifications**: Published Language (Kafka)

### Подробно:
* **Payments -> Wallet: Customer/Supplier (gRPC)**
  Проверка баланса - это критическая операция с высокими требованиями к задержкам (latency). gRPC быстрее сериализуется и поддерживает постоянное HTTP/2 соединение, что снизит нагрузку на сеть между критичными микросервисами.
* **Payments -> Anti-Fraud: Customer/Supplier (gRPC)**
  Синхронная проверка перед списанием средств (блокирующая операция).
* **Payments -> Kafka -> ACL Service Adapter -> Core Banking: Anti-Corruption Layer (Kafka + SOAP)**
  Изоляция от legacy через kafka + адаптер, преобразующий события в формат Core Banking Legacy. Прослойка на Spring Boot может трансформировать события из Kafka в требуемые для Legacy форматы (json->xml).
* **Payments -> Notifications: Published Language (Kafka)**
  Асинхронные события для уведомлений.
* **Kafka -> Callbacks -> Merchant (REST)**
  Отвечает за отправку уведомлений внешним мерчантам о статусе платежей. После завершения платежа (успешного или неуспешного) система должна уведомить мерчанта, чтобы он мог:
    * Завершить процесс продажи
    * Обновить статус заказа
    * Предоставить товар/услугу
    * Обработать ошибки

---

## Брокер

Была выбрана **Kafka**. Используется как шина событий: сервис кошелька, уведомлений и бухгалтерский мс (через адаптер) подписываются на события транзакции.

Также используется в паттерне **Transactional Outbox** (сообщение о платеже сначала попадает в БД платежей, а затем в Kafka). Если Core Banking недоступен, сообщение будет лежать в очереди, пока Адаптер не сможет его успешно передать. Данные не теряются.
