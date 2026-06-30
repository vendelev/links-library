# Clean Architecture, DDD, CQRS, Event-Driven Architecture

Гайд для Lead/Senior PHP backend-разработчика, который готовится к архитектурным интервью.

Цель: уметь не только назвать паттерны, но и объяснить, где они дают пользу, где создают лишнюю сложность, как применяются в Laravel/Symfony/PHP и какие компромиссы ожидаются от senior-инженера.

## Как отвечать на интервью

Хороший senior-ответ обычно содержит:

- определение простыми словами;
- какую проблему решает подход;
- когда применять и когда не применять;
- пример из PHP/Laravel/Symfony;
- tradeoff: цена, сложность, риски;
- связь с тестируемостью, эволюцией системы и командной работой.

Пример короткого ответа:

> Clean Architecture помогает изолировать бизнес-правила от фреймворка, базы данных и UI. В центре находятся entities и use cases, а Laravel/Symfony остаются во внешнем слое. Это повышает тестируемость и снижает vendor lock-in, но добавляет больше классов, DTO и маппинга, поэтому для CRUD-модулей может быть избыточно.

## Clean Architecture

Clean Architecture строит систему вокруг бизнес-правил, а не вокруг фреймворка или базы данных.

Ключевая идея: важная политика приложения должна быть независима от деталей доставки HTTP-запроса, ORM, очередей, брокеров, CLI и UI.

### Слои

Типичная структура:

- Entities: доменные объекты и бизнес-инварианты.
- Use Cases / Interactors: сценарии приложения.
- Interface Adapters: controllers, presenters, gateways, repositories implementations, serializers, mappers.
- Frameworks & Drivers: Laravel, Symfony, Doctrine, Eloquent, PostgreSQL, Redis, RabbitMQ, HTTP, CLI.

Пример для PHP-проекта:

```text
src/
  Domain/
    Order/
      Order.php
      OrderId.php
      OrderStatus.php
      OrderRepository.php
  Application/
    Order/
      PlaceOrderCommand.php
      PlaceOrderHandler.php
  Infrastructure/
    Persistence/
      DoctrineOrderRepository.php
  UI/
    Http/
      PlaceOrderController.php
```

### Dependency Rule

Зависимости направлены внутрь.

- Domain не знает про Application, Infrastructure, Laravel/Symfony, Doctrine/Eloquent.
- Application знает про Domain и абстракции портов.
- Infrastructure реализует интерфейсы, объявленные во внутренних слоях.
- UI вызывает application use cases.

Плохой признак:

```php
final class Order
{
    public function save(): void
    {
        DB::table('orders')->insert([...]);
    }
}
```

Лучше:

```php
interface OrderRepository
{
    public function save(Order $order): void;
}

final class PlaceOrderHandler
{
    public function __construct(private OrderRepository $orders) {}

    public function __invoke(PlaceOrderCommand $command): OrderId
    {
        $order = Order::place($command->customerId, $command->items);
        $this->orders->save($order);

        return $order->id();
    }
}
```

### Entities

Entity в Clean Architecture не обязательно равна DDD Entity. Это объект с бизнес-правилами, который не должен зависеть от технических деталей.

Пример:

```php
final class Subscription
{
    public function cancel(DateTimeImmutable $now): void
    {
        if ($this->status === SubscriptionStatus::Expired) {
            throw new DomainException('Expired subscription cannot be cancelled.');
        }

        $this->status = SubscriptionStatus::Cancelled;
        $this->cancelledAt = $now;
    }
}
```

### Use Cases

Use case описывает конкретный сценарий: `RegisterUser`, `PlaceOrder`, `CancelSubscription`, `ApprovePayout`.

Use case:

- валидирует application-level условия;
- загружает агрегаты;
- вызывает доменную логику;
- сохраняет изменения;
- публикует события или планирует side effects;
- управляет транзакцией на уровне приложения.

Не стоит превращать use case в огромный god service. Если внутри много бизнес-решений, часть логики должна перейти в доменную модель или domain service.

### Interface Adapters

Interface adapters переводят данные между внешним миром и внутренней моделью.

Примеры:

- HTTP controller преобразует request в command DTO.
- Presenter превращает result в JSON/view model.
- Repository implementation маппит ORM model в domain object.
- Messenger/Queue handler вызывает application handler.

Laravel controller:

```php
final class PlaceOrderController
{
    public function __invoke(Request $request, PlaceOrderHandler $handler): JsonResponse
    {
        $orderId = $handler(new PlaceOrderCommand(
            customerId: CustomerId::fromString($request->string('customer_id')),
            items: OrderItemsData::fromArray($request->array('items')),
        ));

        return response()->json(['order_id' => $orderId->toString()], 201);
    }
}
```

Symfony controller:

```php
final class PlaceOrderController
{
    public function __invoke(Request $request, MessageBusInterface $bus): JsonResponse
    {
        $bus->dispatch(new PlaceOrderCommand(
            customerId: $request->request->getString('customer_id'),
            items: $request->request->all('items'),
        ));

        return new JsonResponse(null, 202);
    }
}
```

### Frameworks & Drivers

Laravel, Symfony, Doctrine, Eloquent, Redis, Kafka, RabbitMQ, PostgreSQL и HTTP являются деталями. Они важны, но не должны определять доменную модель.

Практичный senior-подход: не пытаться полностью скрыть фреймворк везде. Изолировать нужно те части, где ожидается сложная бизнес-логика, долгий жизненный цикл и высокая стоимость изменений.

### Частые ошибки

- Делать Clean Architecture ради простого CRUD.
- Называть папки `Domain/Application/Infrastructure`, но оставлять бизнес-логику в controllers/jobs/listeners.
- Тащить `Request`, `Model`, `EntityManager`, `Container`, `DB` во внутренние слои.
- Создавать интерфейс для каждого класса без реальной причины.
- Путать DTO, Entity, ORM model и API resource.
- Считать, что Clean Architecture требует отказа от Laravel/Symfony возможностей.

### Tradeoffs

Плюсы:

- выше тестируемость бизнес-логики;
- проще менять delivery mechanism: HTTP, CLI, queue;
- ниже coupling с фреймворком;
- лучше подходит для больших команд и долгоживущих доменов.

Минусы:

- больше boilerplate;
- сложнее onboarding;
- нужен discipline, иначе получится layered spaghetti;
- mapping между ORM и domain может быть дорогим.

## Hexagonal Architecture / Ports and Adapters

Hexagonal Architecture похожа на Clean Architecture, но описывает систему через порты и адаптеры.

- Port: интерфейс, через который приложение взаимодействует с внешним миром.
- Adapter: конкретная реализация порта.
- Driving adapter: инициирует use case, например HTTP controller, CLI command, queue consumer.
- Driven adapter: вызывается приложением, например repository, email sender, payment gateway.

Пример портов:

```php
interface PaymentGateway
{
    public function charge(Money $amount, PaymentMethodId $methodId): PaymentId;
}

interface InvoiceRepository
{
    public function get(InvoiceId $id): Invoice;
    public function save(Invoice $invoice): void;
}
```

Адаптеры:

```php
final class StripePaymentGateway implements PaymentGateway
{
    public function charge(Money $amount, PaymentMethodId $methodId): PaymentId
    {
        // Stripe SDK call here.
    }
}

final class DoctrineInvoiceRepository implements InvoiceRepository
{
    // Doctrine-specific mapping here.
}
```

На интервью можно сказать:

> Hexagonal Architecture позволяет тестировать application core через порты, подменяя внешние зависимости fake/in-memory адаптерами. Это особенно полезно для платежей, email, очередей, внешних API и persistence.

## DDD Basics

DDD нужен не для красивых классов, а для борьбы со сложностью предметной области.

Главный фокус DDD: модель должна отражать язык бизнеса и защищать бизнес-инварианты.

### Ubiquitous Language

Ubiquitous Language — общий язык команды и domain experts.

Если бизнес говорит `booking`, `settlement`, `payout`, `claim`, не стоит в коде использовать `thing`, `record`, `data`, `processItem`.

Senior-мысль:

> DDD начинается не с Entity и Repository, а с языка, границ контекста и понимания правил бизнеса.

### Entity

Entity имеет identity и жизненный цикл. Две entity могут иметь одинаковые поля, но быть разными объектами.

```php
final class User
{
    public function __construct(
        private UserId $id,
        private Email $email,
    ) {}

    public function changeEmail(Email $email): void
    {
        if ($this->email->equals($email)) {
            return;
        }

        $this->email = $email;
    }
}
```

### Value Object

Value Object не имеет identity, сравнивается по значению и обычно immutable.

```php
final readonly class Money
{
    public function __construct(
        public int $amount,
        public string $currency,
    ) {
        if ($amount < 0) {
            throw new InvalidArgumentException('Amount must be non-negative.');
        }
    }

    public function add(self $other): self
    {
        if ($this->currency !== $other->currency) {
            throw new DomainException('Currency mismatch.');
        }

        return new self($this->amount + $other->amount, $this->currency);
    }
}
```

Хорошие Value Objects: `Email`, `Money`, `OrderId`, `DateRange`, `Percent`, `Quantity`, `Slug`.

### Aggregate

Aggregate — группа объектов, которая изменяется как единая consistency boundary.

Внутри aggregate root защищает инварианты. Внешний код не должен менять внутренние entity напрямую.

Пример:

```php
final class Order
{
    /** @var list<OrderLine> */
    private array $lines = [];

    public function addItem(ProductId $productId, Quantity $quantity, Money $price): void
    {
        if ($this->status !== OrderStatus::Draft) {
            throw new DomainException('Only draft order can be changed.');
        }

        $this->lines[] = new OrderLine($productId, $quantity, $price);
    }

    public function submit(): void
    {
        if ($this->lines === []) {
            throw new DomainException('Empty order cannot be submitted.');
        }

        $this->status = OrderStatus::Submitted;
    }
}
```

### Aggregate Root

Aggregate Root — единственная точка входа в aggregate.

Правило:

- repository работает с aggregate root;
- внешние объекты хранят ссылку на root id, а не на внутренние entity;
- транзакционная консистентность гарантируется внутри одного aggregate;
- между aggregates чаще используется eventual consistency.

### Repository

Repository абстрагирует получение и сохранение aggregate.

Это не generic DAO и не место для бизнес-логики.

```php
interface OrderRepository
{
    public function get(OrderId $id): Order;
    public function save(Order $order): void;
}
```

Плохой smell:

```php
$orderRepository->markAsPaidAndSendEmailAndCreateInvoice($id);
```

Лучше: application service orchestration плюс domain methods.

### Domain Service

Domain Service нужен, когда бизнес-операция не принадлежит естественно одной entity/value object.

Пример: расчет комиссии зависит от merchant, региона, категории и правил тарифа.

```php
final class CommissionCalculator
{
    public function calculate(Merchant $merchant, Money $amount, Region $region): Money
    {
        // Domain rules here.
    }
}
```

Не стоит называть domain service любой service-класс. Если он просто вызывает repository и queue, это application service.

### Application Service

Application Service координирует use case, но не содержит сложных доменных правил.

```php
final class PayInvoiceHandler
{
    public function __construct(
        private InvoiceRepository $invoices,
        private PaymentGateway $payments,
        private EventBus $events,
    ) {}

    public function __invoke(PayInvoiceCommand $command): void
    {
        $invoice = $this->invoices->get($command->invoiceId);
        $paymentId = $this->payments->charge($invoice->total(), $command->paymentMethodId);

        $invoice->markAsPaid($paymentId);

        $this->invoices->save($invoice);
        $this->events->publish(...$invoice->releaseEvents());
    }
}
```

### Domain Event

Domain Event фиксирует факт, который уже произошел в домене.

Имена обычно в прошедшем времени: `OrderPlaced`, `InvoicePaid`, `UserRegistered`, `ShipmentDispatched`.

```php
final readonly class InvoicePaid
{
    public function __construct(
        public InvoiceId $invoiceId,
        public PaymentId $paymentId,
        public DateTimeImmutable $paidAt,
    ) {}
}
```

Domain Event не должен быть командой. Он не говорит `SendEmail`; он говорит `InvoicePaid`.

### Bounded Context

Bounded Context — граница, внутри которой модель и язык имеют точный смысл.

Один и тот же термин может означать разное:

- `Customer` в Sales: потенциальный покупатель.
- `Customer` в Billing: юридическое лицо для выставления счета.
- `Customer` в Support: пользователь с тикетами и SLA.

Контексты можно связать через:

- published language;
- anti-corruption layer;
- events;
- REST/RPC API;
- shared kernel, если команда осознанно принимает tight coupling.

### DDD Pitfalls

- Anemic domain model: вся логика в services, entity только getters/setters.
- Огромные aggregates, которые блокируют производительность и параллелизм.
- Repository на каждую таблицу, а не на aggregate root.
- Value Objects без поведения.
- Domain Events используются как технические queue jobs.
- Bounded Contexts рисуются по таблицам или командам разработки, а не по языку и правилам бизнеса.
- Слишком раннее внедрение DDD в CRUD-продукт без сложной доменной логики.

## CQRS

CQRS разделяет модель записи и модель чтения.

- Command меняет состояние.
- Query возвращает данные и не меняет состояние.
- Write model защищает инварианты.
- Read model оптимизирована под отображение и поиск.

CQRS не требует Event Sourcing и микросервисов.

### Commands

Command выражает намерение пользователя или системы.

```php
final readonly class ApproveWithdrawalCommand
{
    public function __construct(
        public WithdrawalId $withdrawalId,
        public AdminId $approvedBy,
    ) {}
}
```

Command handler обычно ничего не возвращает или возвращает id/result, но не read model.

### Queries

Query возвращает данные без side effects.

```php
final readonly class FindOrdersForCustomerQuery
{
    public function __construct(
        public CustomerId $customerId,
        public int $limit = 50,
    ) {}
}
```

Query handler может использовать SQL напрямую, read-optimized tables, Elasticsearch, Redis или materialized views.

```php
final class FindOrdersForCustomerHandler
{
    public function __construct(private Connection $db) {}

    /** @return list<OrderListItem> */
    public function __invoke(FindOrdersForCustomerQuery $query): array
    {
        return $this->db->fetchAllAssociative('SELECT id, status, total FROM order_read_model WHERE customer_id = ?', [
            $query->customerId->toString(),
        ]);
    }
}
```

### Read Models

Read model — структура данных, удобная для чтения.

Примеры:

- таблица `order_list_view` для списка заказов;
- денормализованная таблица dashboard metrics;
- Elasticsearch index для поиска;
- Redis cache для быстрых counters.

Read model может быть eventually consistent, если обновляется асинхронно из событий.

### Eventual Consistency

Eventual consistency означает, что после записи read model может обновиться не мгновенно.

На интервью важно объяснить UX и product implications:

- пользователь может увидеть старый статус несколько секунд;
- UI может показывать `processing`;
- нужны retries и monitoring lag;
- критичные команды должны проверять write model, а не read model.

### CQRS в Symfony и Laravel

Symfony:

- `symfony/messenger` для command bus/event bus/async handlers;
- Doctrine для write model;
- DBAL или read repositories для queries.

Laravel:

- Jobs/Queues для async commands/events;
- Events/Listeners для реакций;
- Action classes или command handlers для use cases;
- Eloquent можно оставить для read side, если это практично.

### CQRS Pitfalls

- Делать CQRS для всех CRUD-экранов.
- Путать command с HTTP request DTO.
- Делать query через domain aggregates, когда нужен простой read projection.
- Возвращать из command handler огромные read DTO.
- Не учитывать eventual consistency в UX и тестах.
- Не иметь стратегии rebuild read models.

## Event-Driven Architecture

Event-Driven Architecture строит взаимодействие компонентов вокруг событий.

Событие говорит, что что-то уже произошло. Подписчики реагируют независимо.

Пример: `OrderPlaced` может привести к резервированию товара, отправке письма, начислению бонусов и обновлению аналитики.

### Events vs Messages

Message — общий контейнер передачи данных.

Event — тип сообщения, описывающий факт прошлого.

Command message — просьба выполнить действие.

```text
Command: ShipOrder
Event: OrderShipped
Message: транспортная оболочка с payload, headers, correlation_id, causation_id
```

Senior-ответ:

> Event не должен знать, кто его обработает. Если отправитель ожидает конкретное действие от конкретного получателя, это скорее command, а не event.

### Brokers

Типичные брокеры:

- RabbitMQ: routing, queues, acknowledgements, retry/dead-letter, хорош для task/event messaging.
- Kafka: distributed log, partitions, consumer groups, replay, высокая throughput, event streaming.
- Redis Streams: проще, но меньше гарантий и tooling по сравнению с Kafka.
- AWS SNS/SQS/EventBridge: managed messaging/event routing.

Что важно обсудить:

- delivery semantics: at-most-once, at-least-once, effectively-once;
- ordering: глобальный порядок почти никогда не нужен, порядок обычно в рамках aggregate id/partition key;
- retries и dead-letter queues;
- schema evolution;
- observability: correlation id, tracing, lag, failed messages.

### Outbox Pattern

Проблема: нужно атомарно сохранить состояние в БД и отправить событие в брокер.

Нельзя надежно сделать так:

```php
$orderRepository->save($order);
$eventBus->publish(new OrderPlaced($order->id()));
```

Если процесс упадет между save и publish, событие потеряется.

Outbox pattern:

- в той же БД-транзакции сохраняем aggregate и запись в `outbox_messages`;
- отдельный publisher читает outbox и отправляет в брокер;
- после успешной отправки помечает запись как published;
- consumers должны быть idempotent, потому что возможны дубли.

Схема:

```text
Application transaction:
  update orders
  insert outbox_messages

Publisher worker:
  select unpublished messages
  publish to broker
  mark as published
```

Laravel/Symfony пример: outbox publisher можно реализовать как scheduled command/worker, а публикацию делать через Symfony Messenger или Laravel Queue/Event.

### Inbox and Idempotency

At-least-once delivery означает, что consumer может получить одно событие несколько раз.

Consumer должен быть idempotent.

Inbox pattern:

- каждое входящее сообщение имеет `message_id`;
- consumer перед обработкой проверяет `processed_messages`;
- обработка и запись `processed_messages` выполняются в одной транзакции;
- повторное сообщение игнорируется.

Пример:

```php
public function __invoke(OrderPlaced $event): void
{
    $this->transactional->run(function () use ($event): void {
        if ($this->inbox->alreadyProcessed($event->messageId)) {
            return;
        }

        $this->bonusService->accrueForOrder($event->orderId);
        $this->inbox->markProcessed($event->messageId);
    });
}
```

Idempotency также нужна для HTTP commands, например платежей. Часто используется `Idempotency-Key`.

### Saga / Process Manager

Saga управляет долгим бизнес-процессом, который затрагивает несколько aggregates или сервисов.

Пример checkout:

1. OrderPlaced.
2. ReserveInventory.
3. ChargePayment.
4. CreateShipment.
5. Если payment failed, release inventory и mark order as payment failed.

Process Manager хранит состояние процесса и решает, какую command отправить дальше.

```text
OrderPlaced -> CheckoutProcessManager -> ReserveInventory
InventoryReserved -> CheckoutProcessManager -> ChargePayment
PaymentFailed -> CheckoutProcessManager -> ReleaseInventory
```

Важно: saga не дает ACID-транзакцию между сервисами. Она дает orchestrated eventual consistency и compensating actions.

### Event Sourcing Basics

Event Sourcing хранит состояние как последовательность событий, а не как текущий snapshot.

Вместо:

```text
orders: id, status, total
```

Храним:

```text
OrderPlaced
OrderItemAdded
OrderSubmitted
OrderPaid
OrderShipped
```

Текущее состояние восстанавливается replay событий.

Плюсы:

- полный audit trail;
- возможность replay и построения новых projections;
- хорошо подходит для финансовых, workflow и compliance-доменов;
- естественная связь с CQRS.

Минусы:

- сложность schema evolution;
- сложность debugging для команды без опыта;
- eventual consistency почти неизбежна;
- нужны snapshots для больших streams;
- нельзя легко изменить прошлое событие.

PHP-библиотеки:

- EventSauce: pragmatical event sourcing для PHP.
- Prooph: исторически популярный набор компонентов, сейчас стоит проверять актуальность и поддержку перед выбором.

### Event-Driven Pitfalls

- Использовать events как скрытые synchronous function calls.
- Слишком мелкие события без бизнес-смысла: `FieldChanged` вместо `InvoicePaid`.
- Отсутствие idempotency.
- Нет DLQ, retry policy и monitoring.
- Нет версионирования event schema.
- Event payload содержит внутреннюю ORM-модель или нестабильный JSON.
- Сильная связность через shared database.
- Непонимание ordering guarantees брокера.

## Modular Monolith vs Microservices

### Modular Monolith

Modular monolith — один deployable artifact, но код разделен на модули с явными границами.

Пример:

```text
src/
  Billing/
    Domain/
    Application/
    Infrastructure/
    UI/
  Catalog/
  Ordering/
  Shipping/
```

Плюсы:

- проще разработка, деплой и локальный запуск;
- проще транзакции;
- меньше distributed systems complexity;
- хорошо подходит для команды, которая еще ищет границы домена.

Минусы:

- нужна дисциплина границ;
- риск превратиться в big ball of mud;
- масштабирование только всего приложения, если не выделять процессы отдельно.

Практичный ответ:

> Я часто предпочту modular monolith как стартовую архитектуру. Он позволяет применять DDD boundaries без преждевременной цены микросервисов. Если границы стабилизировались и есть независимые scaling/deployment needs, модуль можно выделять.

### Microservices

Microservices полезны, когда есть:

- независимые команды и ownership;
- разные требования по масштабированию;
- независимые релизы;
- высокая цена coupling в монолите;
- зрелая платформа: CI/CD, observability, tracing, incident response.

Цена микросервисов:

- network failures;
- distributed transactions отсутствуют или нежелательны;
- eventual consistency;
- сложнее тестирование;
- versioning контрактов;
- observability обязательна;
- больше DevOps/SRE overhead.

Senior-позиция:

> Микросервисы не решают плохие границы домена, они делают плохие границы сетевой проблемой.

## Strangler Fig Pattern

Strangler Fig помогает постепенно заменить legacy-систему.

Идея: новый функционал или migrated slices строятся рядом со старой системой, а маршрутизация постепенно переводится на новую реализацию.

Шаги:

1. Найти seam: endpoint, module, table boundary, business capability.
2. Поставить routing/proxy/facade.
3. Реализовать новый slice отдельно.
4. Синхронизировать данные через API/events/CDC/dual-write с осторожностью.
5. Перевести трафик.
6. Удалить legacy-код.

PHP-примеры:

- старый Laravel controller вызывает новый application service для одного use case;
- Symfony reverse proxy/router направляет часть endpoint в новый сервис;
- legacy Eloquent model постепенно заменяется repository adapter;
- новый bounded context читает legacy DB через anti-corruption layer.

Риски:

- dual-write без outbox может потерять события;
- слишком долгий transition period создает две системы поддержки;
- нет метрик parity между old/new behavior;
- команда начинает переписывать все сразу вместо thin vertical slices.

## Практические объяснения для интервью

### Clean Architecture за 30 секунд

Clean Architecture отделяет бизнес-правила от деталей. Внутри находятся domain entities и use cases, снаружи — controllers, ORM, queues, фреймворк. Зависимости направлены внутрь. Это повышает тестируемость и делает систему устойчивее к изменению инфраструктуры, но добавляет boilerplate и не всегда нужно для простого CRUD.

### DDD за 30 секунд

DDD помогает моделировать сложную предметную область через общий язык, bounded contexts, aggregates, value objects и domain events. Его цель — не паттерны ради паттернов, а код, который выражает бизнес-правила и защищает инварианты.

### CQRS за 30 секунд

CQRS разделяет write side и read side. Commands меняют состояние и защищают инварианты, queries читают оптимизированные read models. Это полезно при сложных сценариях записи, тяжелых read views и разных требованиях к производительности, но добавляет eventual consistency и operational complexity.

### Event-Driven за 30 секунд

Event-Driven Architecture связывает компоненты через события о произошедших фактах. Это снижает coupling и помогает масштабировать реакции на бизнес-события, но требует idempotency, retries, DLQ, schema evolution, observability и понимания delivery guarantees.

### Modular Monolith за 30 секунд

Modular monolith дает модульные границы и DDD-подход без сетевой сложности микросервисов. Это часто лучший старт для продукта. Микросервисы стоит выделять позже, когда границы стабильны и есть реальные причины для независимого scaling/deployment.

## Mini-Practice: Refactoring Examples

### 1. Fat Controller в Use Case

До:

```php
public function store(Request $request): JsonResponse
{
    $order = Order::create($request->all());
    Mail::to($order->user)->send(new OrderCreatedMail($order));
    dispatch(new SyncOrderToCrm($order->id));

    return response()->json($order);
}
```

Проблемы:

- controller содержит orchestration;
- Eloquent model выступает и persistence model, и domain model;
- side effects смешаны с HTTP;
- трудно тестировать бизнес-сценарий без framework integration.

После:

```php
final class CreateOrderHandler
{
    public function __construct(
        private OrderRepository $orders,
        private EventBus $events,
    ) {}

    public function __invoke(CreateOrderCommand $command): OrderId
    {
        $order = Order::create($command->customerId, $command->items);

        $this->orders->save($order);
        $this->events->publish(new OrderCreated($order->id()));

        return $order->id();
    }
}
```

Controller только собирает command и возвращает response.

### 2. Primitive Obsession в Value Object

До:

```php
$user->changeEmail($request->input('email'));
```

После:

```php
$user->changeEmail(Email::fromString($request->input('email')));
```

Польза:

- email валидируется в одном месте;
- тип выражает смысл;
- проще тестировать;
- меньше duplicated validation.

### 3. Synchronous Side Effect в Outbox

До:

```php
$this->orders->save($order);
$this->messageBus->dispatch(new OrderPlaced($order->id()));
```

После:

```php
$this->transactional->run(function () use ($order): void {
    $this->orders->save($order);
    $this->outbox->add(new OrderPlaced($order->id()));
});
```

Publisher отправит событие позже, надежно и с retry.

### 4. Query через Aggregate в Read Model

До:

```php
$orders = array_map(
    fn (OrderId $id) => $this->orders->get($id),
    $this->orderIndex->findIdsByCustomer($customerId),
);
```

После:

```php
$orders = $this->db->fetchAllAssociative(
    'SELECT id, status, total, created_at FROM order_read_model WHERE customer_id = ? ORDER BY created_at DESC LIMIT 50',
    [$customerId->toString()],
);
```

Для read use case не всегда нужна domain model.

## Common Senior-Level Tradeoffs

### Когда не применять Clean Architecture

- простой CRUD без сложных правил;
- короткоживущий internal tool;
- маленькая команда без архитектурной дисциплины;
- MVP, где важнее проверить гипотезу.

Но можно применить частично: выделить use cases для сложных сценариев и оставить простые CRUD на framework conventions.

### Когда DDD оправдан

- сложные бизнес-правила;
- много исключений и инвариантов;
- разные отделы используют разные смыслы одних терминов;
- частые изменения правил;
- высокая цена ошибки в домене.

### Когда CQRS оправдан

- write и read модели сильно отличаются;
- сложные read projections;
- высокие нагрузки на чтение;
- нужны async projections;
- read side должен быть денормализован.

### Когда Event-Driven оправдан

- много независимых реакций на бизнес-события;
- интеграции с другими системами;
- нужно decoupling между bounded contexts;
- долгие процессы;
- аудит и replay важны.

### Когда микросервисы оправданы

- независимые команды;
- независимый deploy;
- разные scaling profiles;
- fault isolation;
- bounded contexts уже понятны.

## Архитектурный Review Checklist

### Границы и зависимости

- Бизнес-логика не живет в controllers/listeners/jobs?
- Domain не зависит от Laravel/Symfony/Doctrine/Eloquent?
- Use cases выражают бизнес-сценарии, а не технические операции?
- Infrastructure реализует порты, а не протекает внутрь?
- Есть ли циклические зависимости между модулями?

### DDD

- Термины в коде совпадают с языком бизнеса?
- Aggregates защищают инварианты?
- Repository работает с aggregate root?
- Value Objects immutable и содержат поведение?
- Domain Services не стали dumping ground?
- Bounded Contexts имеют явные границы и contracts?

### CQRS

- Commands не возвращают read models?
- Queries не имеют side effects?
- Read model можно rebuild?
- Eventual consistency понятна product/UX?
- Критичные decisions читают write model, а не stale projection?

### Event-Driven

- Events названы в прошедшем времени и отражают бизнес-факт?
- Есть outbox для надежной публикации?
- Consumers idempotent?
- Есть inbox/processed messages для external events?
- Настроены retries, DLQ, monitoring?
- Есть correlation id и causation id?
- Есть стратегия версионирования event schema?

### Operations

- Есть metrics, logs, tracing?
- Видно queue lag и failed messages?
- Есть replay/rebuild procedures?
- Тестируются failure scenarios?
- Понятен rollback plan?

## Self-Check Questions

1. Почему dependency rule важнее названий папок?
2. Чем use case отличается от controller action?
3. Когда repository interface стоит держать в Domain, а когда это лишнее?
4. Чем Entity отличается от Value Object?
5. Почему aggregate boundary не должен быть слишком большим?
6. Чем Domain Service отличается от Application Service?
7. Почему Domain Event должен быть в прошедшем времени?
8. Чем Bounded Context отличается от модуля или микросервиса?
9. Можно ли использовать CQRS без Event Sourcing?
10. Почему read model может быть денормализованной?
11. Что сломается, если пользователь увидит stale read model?
12. Почему outbox pattern нужен при публикации событий?
13. Почему consumer должен быть idempotent?
14. Чем saga отличается от distributed transaction?
15. Когда Kafka лучше RabbitMQ, а когда наоборот?
16. Что такое schema evolution для events?
17. Почему микросервисы не должны быть первым шагом при непонятных границах?
18. Как Strangler Fig снижает риск legacy migration?
19. Где в Laravel/Symfony лучше держать application handlers?
20. Как протестировать use case без HTTP и базы данных?

## Короткие ответы Senior-уровня

### Почему не Active Record в Domain?

Active Record смешивает domain behavior и persistence. В Laravel это удобно для CRUD, но в сложном домене может привести к leakage базы данных в бизнес-логику. Практично использовать Eloquent на read side или как persistence adapter, а сложные aggregates держать отдельно.

### Нужно ли всегда делать интерфейс для repository?

Нет. Интерфейс полезен, когда он является портом внутреннего слоя, нужен для тестирования или есть несколько реализаций. Интерфейс ради интерфейса увеличивает boilerplate.

### Чем event отличается от command?

Command выражает намерение выполнить действие и обычно имеет одного logical receiver. Event описывает факт, который уже произошел, и может иметь много независимых subscribers.

### CQRS всегда означает две базы?

Нет. CQRS означает разделение моделей чтения и записи. Это могут быть разные классы и SQL-запросы в одной базе. Отдельные хранилища появляются только при реальной необходимости.

### Event Sourcing нужен для Event-Driven Architecture?

Нет. Event-Driven может публиковать события из обычной state-based модели. Event Sourcing — отдельный паттерн хранения состояния как журнала событий.

### Можно ли использовать DDD в Laravel?

Да. Laravel можно оставить во внешнем слое: controllers, queues, service providers, Eloquent adapters. Важно не тащить facades/request/model в domain core там, где нужна изоляция.

### Почему modular monolith часто лучше микросервисов?

Он дает границы и модульность без distributed systems complexity. Если границы окажутся неверными, исправить их внутри монолита дешевле, чем между сервисами с контрактами, данными и сетью.

## Ссылки

- Martin Fowler: Patterns of Enterprise Application Architecture — https://martinfowler.com/books/eaa.html
- Martin Fowler: CQRS — https://martinfowler.com/bliki/CQRS.html
- Martin Fowler: Event Sourcing — https://martinfowler.com/eaaDev/EventSourcing.html
- Martin Fowler: Strangler Fig Application — https://martinfowler.com/bliki/StranglerFigApplication.html
- Martin Fowler: Bounded Context — https://martinfowler.com/bliki/BoundedContext.html
- Robert C. Martin: Clean Architecture article — https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html
- Robert C. Martin: Clean Architecture book — https://www.oreilly.com/library/view/clean-architecture-a/9780134494272/
- Alistair Cockburn: Hexagonal Architecture — https://alistair.cockburn.us/hexagonal-architecture/
- Eric Evans: Domain-Driven Design — https://www.domainlanguage.com/ddd/
- Vaughn Vernon: Implementing Domain-Driven Design — https://vaughnvernon.co/?page_id=168
- Vaughn Vernon: Domain-Driven Design Distilled — https://vaughnvernon.co/?page_id=168
- Designing Data-Intensive Applications, Martin Kleppmann — https://dataintensive.net/
- Microsoft: CQRS pattern — https://learn.microsoft.com/en-us/azure/architecture/patterns/cqrs
- Microsoft: Event Sourcing pattern — https://learn.microsoft.com/en-us/azure/architecture/patterns/event-sourcing
- Microsoft: Saga distributed transactions pattern — https://learn.microsoft.com/en-us/azure/architecture/reference-architectures/saga/saga
- AWS Prescriptive Guidance: Strangler Fig pattern — https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/strangler-fig.html
- AWS Prescriptive Guidance: Transactional outbox pattern — https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/transactional-outbox.html
- Symfony Messenger — https://symfony.com/doc/current/messenger.html
- Symfony EventDispatcher — https://symfony.com/doc/current/components/event_dispatcher.html
- Laravel Events — https://laravel.com/docs/events
- Laravel Queues — https://laravel.com/docs/queues
- EventSauce — https://eventsauce.io/
- Prooph Event Store — https://github.com/prooph/event-store
