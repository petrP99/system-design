# description.md

# Архитектура платежной системы (Saga Orchestration & CQRS)

Данный документ описывает целевую архитектуру системы обработки платежей с использованием паттернов **Saga (Оркестрация)**,
**Transactional Outbox (Debezium)** и **CQRS (ClickHouse)**.

---

## 1. Общее описание сценариев

### 1.1 [Успешный платеж](../schemas/sequences/happy-path.puml)
1. **Клиент** отправляет запрос через **Kong API Gateway** в **Orchestrator**.
2. **Orchestrator** вызывает синхронный gRPC запрос в **Wallet Service** (`reserveFunds`).
3. **Wallet Service**:
   * Проверяет баланс и резервирует средства (`available` -> `reserved`).
   * Атомарно обновляет баланс и пишет событие `PaymentInitiated`в таблицу **wallet_outbox**.
   * Возвращает Orchestrator-у ответ "Принято".
4. **Debezium (CDC)** считывает лог **Wallet DB** и публикует `payment.initiated` в **Kafka**.
5. **Transaction Service** получает событие из Kafka, сохраняет тех. запись (`PENDING`) и инициирует запрос к **Внешнему провайдеру**.
6. **Callback Service** принимает Webhook от провайдера, валидирует его и через HTTPS уведомляет **Transaction Service**.
7. **Transaction Service** обновляет тех. статус на `SUCCESS` и через свой **Outbox (Debezium)** шлет `payment.result` в Kafka.
8. **Orchestrator** получает `SUCCESS` и вызывает `Wallet.confirmFunds` для финального списания.
9. **Payment Query Service** слушает все события и обновляет проекцию в **ClickHouse**.

### 1.2 [Неуспешный платеж (Компенсация)](../schemas/sequences/failed-path.puml)
1. Если **Transaction Service** фиксирует отказ провайдера или таймаут, он публикует `payment.result (FAILED)`.
2. **Orchestrator**, получив отказ, запускает шаг компенсации: вызывает `Wallet.releaseFunds`.
3. **Wallet Service** возвращает резерв на доступный баланс и обновляет статус транзакции на `CANCELLED`.
4. **Query Service** фиксирует финальное состояние ошибки в read-модели.

---

### 1.3 Примеры сообщений kafka-events:
* [payment-initiated](kafka-events/paymentInitiated.json)
* [payment-completed](kafka-events/paymentCompleted.json)
* [payment-failed](kafka-events/paymentFailed.json)
* [provider-callback](kafka-events/providerCallback.json)

## 2. Технические решения и паттерны

*   **Оркестрация**: Централизованное управление жизненным циклом платежа через **Orchestrator**.
*   **Transactional Outbox (CDC)**: Использование **Debezium** для гарантированной доставки событий из БД в Kafka без потери данных при сбоях приложения.
*   **Идемпотентность**: Проверка по `payment_id` (UUIDv7) на уровне Unique Constraint в Wallet и Transaction DB.
*   **CQRS**: Полное разделение Write-модели (PostgreSQL) и Read-модели (**ClickHouse**). Чтение истории и статусов не нагружает транзакционные сервисы.

---

## 3. Спецификация API контрактов

### 3.1 Внешние API
* **[Orchestrator](api/orchestrator-bff.http)**:
* * [POST] `/v1/payments`**: Инициация платежа. Принимает `amount`, `target_account`, `idempotency_key`.
* * [GET]`/v1/payments/{payment_id}`**: Получение статуса из Query Service.

### 3.2 Внутренние API (gRPC)
* **[Wallet Service](api/wallet.http)**:
   * `reserveFunds(payment_id, user_id, amount)` — Блокировка средств.
   * `confirmFunds(payment_id)` — Финальное списание.
   * `releaseFunds(payment_id)` — Отмена резерва (Компенсация).
* **[Transaction Service](api/transaction-service.http)**:
   * `processPayment(payment_id, provider_id, amount)` — Вызов провайдера.
* **[Callback](api/callback.http)**:
   * `updateProviderStatus(payment_id, status)` — Прием данных от  Service.

---

## 4. [Схема данных (ERD)](../schemas/erd/db-diagram.mmd)

### Wallet DB (PostgreSQL)
* **wallets**: `id`, `user_id`, `available_balance`, `reserved_balance`.
* **wallet_transactions**: `payment_id` (PK), `wallet_id`, `amount`, `status`.
* **wallet_outbox**: `id`, `event_type`, `payload`.

### Transaction DB (PostgreSQL)
* **payments**: `payment_id` (PK), `external_ref`, `tech_status`, `updated_at`.
* **tx_outbox**: `id`, `event_type`, `payload`.

### Read DB (ClickHouse)
* **payment_view**: `payment_id`, `user_id`, `amount`, `status`, `provider_status`, `created_at`.

---

