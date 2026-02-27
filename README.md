# Архитектура платежной вертикали банка

Репозиторий содержит комплект архитектурной документации по трансформации платежной системы. Проект направлен на
разделение монолитной логики, внедрение событийной модели (EDA) и изоляцию Legacy-систем.

## Описание задачи

Цель проекта — «очеловечивание» платежной платформы за счет:

* Разделения ответственности через **Bounded Contexts**.
* Снижения связности компонентов с помощью **Message Broker**.
* Внедрения **Anti-Corruption Layer (ACL)** для работы с Core Banking.
* Перехода на единые идентификаторы (**UUID**).

---

## Стратегический уровень (DDD)

Центральным артефактом проектирования является **Context Map**, которая определяет границы систем и характер их
взаимодействия.

* **[Описание логики](descriptions/description.md)** — здесь зафиксированы границы контекстов (Wallet, Payments,
  Anti-Fraud и др.) и описаны типы связей (Upstream/Downstream, ACL, Published Language).

---

## Проектирование C4 Model

Иерархические диаграммы системы, выполненные в нотации C4:

1. **[C1: System Context (Уровень системы)](uml-schemas/с1-context.puml)**
   Отображает взаимодействие платежной платформы с внешними потребителями (клиенты) и провайдерами (СБП, Карточный
   процессинг).
2. **[C2: Containers (Уровень контейнеров)](uml-schemas/с2-container.puml)**
   Детализация внутренних сервисов: API Gateway, BFF, микросервисы и шина данных (Kafka).

---

## Технические спецификации

Подробное описание ключевых компонентов и стандартов:

* **[API Gateway](descriptions/api-gateway.md)** — описание работы входного шлюза, функций аутентификации и
  маршрутизации.
* **[UUID Identifiers](descriptions/UUID-identifiers.md)** — стандарт сквозной идентификации операций для обеспечения
  идемпотентности.

---

## Схемы компонентов (PlantUML)

Детальное описание структуры отдельных модулей:

* [Kong Gateway](uml-schemas/components/kong-gateway.puml)
* [Core Banking Integration](uml-schemas/components/core-banking.puml)
* [Payments](uml-schemas/components/payments.puml)
* [Wallets](uml-schemas/components/wallets.puml)
* [Anti-Fraud](uml-schemas/components/anti-fraud.puml)
* [Notifications](uml-schemas/components/notifications.puml) 
* [Callbacks](uml-schemas/components/callbacks.puml)

---

## Ключевые архитектурные решения


1. **Изоляция Legacy:** Прямое обращение к Core Banking запрещено. Весь обмен идет через компонент-обертку (ACL).
2. **Событийная модель:** Нотификации и коллбэки работают асинхронно, подписываясь на события платежного домена.
