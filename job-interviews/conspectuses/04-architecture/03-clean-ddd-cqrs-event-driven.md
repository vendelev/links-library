# Clean Architecture, DDD, CQRS and Event-Driven Architecture

<!-- markdownlint-disable MD013 -->

Конспект для архитектурных интервью senior/lead backend. Фокус: как объяснять архитектурные стили, где они помогают, где создают лишнюю сложность и как применять их в PHP/Laravel/Symfony.

## Как отвечать на интервью

Хороший senior-level ответ содержит:

- определение простыми словами;
- какую проблему решает подход;
- когда применять и когда не применять;
- пример из PHP/Laravel/Symfony;
- trade-off: цена, сложность, риски;
- связь с тестируемостью, эволюцией системы и командной работой.

Короткий пример:

> Clean Architecture помогает изолировать бизнес-правила от фреймворка, базы данных и UI. В центре находятся entities и use cases, а Laravel/Symfony остаются во внешнем слое. Это повышает тестируемость и снижает vendor lock-in, но добавляет больше классов, DTO и mapping, поэтому для CRUD-модулей может быть избыточно.

## Clean Architecture

Clean Architecture строит систему вокруг бизнес-правил, а не вокруг фреймворка или базы данных.

Главная идея: важная политика приложения должна быть независима от деталей доставки HTTP-запроса, ORM, очередей, брокеров, CLI и UI.

### Слои

Типичная структура:

- Entities: доменные объекты и бизнес-инварианты.
- Use Cases / Interactors: сценарии приложения.
- Interface Adapters: controllers, presenters, gateways, repository implementations, serializers, mappers.
- Frameworks & Drivers: Laravel, Symfony, Doctrine, Eloquent, PostgreSQL, Redis, RabbitMQ, HTTP, CLI.

Пример PHP-структуры:

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
        DB::table('orders')->insert([]);
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

### Entities and Use Cases

Entity в Clean Architecture не обязательно равна DDD Entity. Это объект с бизнес-правилами, который не зависит от технических деталей.

Use case описывает конкретный сценарий: `RegisterUser`, `PlaceOrder`, `CancelSubscription`, `ApprovePayout`.

Use case обычно:

- валидирует application-level условия;
- загружает aggregates;
- вызывает domain logic;
- сохраняет изменения;
- публикует события или планирует side effects;
- управляет транзакцией на уровне приложения.

Не стоит превращать use case в God Service. Если внутри много бизнес-решений, часть логики должна перейти в domain model или domain service.

### Interface Adapters

Interface adapters переводят данные между внешним миром и внутренней моделью.

Примеры:

- HTTP controller преобразует request в command DTO.
- Presenter превращает result в JSON/view model.
- Repository implementation маппит ORM model в domain object.
- Messenger/Queue handler вызывает application handler.

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

### Frameworks and Drivers

Laravel, Symfony, Doctrine, Eloquent, Redis, Kafka, RabbitMQ, PostgreSQL и HTTP являются деталями. Они важны, но не должны определять domain model.

Практичный senior-подход: не пытаться полностью скрыть фреймворк везде. Изолировать нужно части, где есть сложная бизнес-логика, долгий жизненный цикл и высокая стоимость изменений.

### Clean Architecture Pitfalls

- Делать Clean Architecture ради простого CRUD.
- Называть папки `Domain/Application/Infrastructure`, но оставлять бизнес-логику в controllers/jobs/listeners.
- Тащить `Request`, `Model`, `EntityManager`, `Container`, `DB` во внутренние слои.
- Создавать интерфейс для каждого класса без причины.
- Путать DTO, Entity, ORM model и API resource.
- Считать, что Clean Architecture требует отказа от Laravel/Symfony возможностей.

### Trade-Offs

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

Hexagonal Architecture похожа на Clean Architecture, но описывает систему через ports и adapters.

- Port: интерфейс, через который приложение взаимодействует с внешним миром.
- Adapter: конкретная реализация порта.
- Driving adapter: инициирует use case, например HTTP controller, CLI command, queue consumer.
- Driven adapter: вызывается приложением, например repository, email sender, payment gateway.

Пример ports:

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

Пример adapters:

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
    // Doctrine mapping here.
}
```

На интервью:

> Hexagonal Architecture позволяет тестировать application core через ports, подменяя внешние зависимости fake/in-memory adapters. Это особенно полезно для платежей, email, очередей, внешних API и persistence.

## DDD Basics

DDD нужен не для красивых классов, а для борьбы со сложностью предметной области.

Главный фокус: модель должна отражать язык бизнеса и защищать бизнес-инварианты.

### Ubiquitous Language

Ubiquitous Language - общий язык команды и domain experts.

Если бизнес говорит `booking`, `settlement`, `payout`, `claim`, не стоит в коде использовать `thing`, `record`, `data`, `processItem`.

DDD начинается не с Entity и Repository, а с языка, границ контекста и понимания правил бизнеса.

### Entity

Entity имеет identity и lifecycle. Две entity могут иметь одинаковые поля, но быть разными объектами.

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

### Aggregate and Aggregate Root

Aggregate - группа объектов, которая изменяется как единая consistency boundary.

Aggregate Root - единственная точка входа в aggregate.

Правила:

- repository работает с aggregate root;
- внешний код не меняет внутренние entities напрямую;
- внешние объекты хранят ссылку на root id, а не на внутренние entities;
- транзакционная консистентность гарантируется внутри одного aggregate;
- между aggregates чаще используется eventual consistency.

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

### Repository

Repository абстрагирует получение и сохранение aggregate. Это не generic DAO и не место для бизнес-логики.

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

Application Service координирует use case, но не содержит сложных domain rules.

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

Bounded Context - граница, внутри которой модель и язык имеют точный смысл.

Один термин может означать разное:

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

- Anemic domain model: вся логика в services, entities только getters/setters.
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

CQRS не требует Event Sourcing, микросервисов или двух баз данных.

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
        return $this->db->fetchAllAssociative(
            'SELECT id, status, total FROM order_read_model WHERE customer_id = ?',
            [$query->customerId->toString()],
        );
    }
}
```

### Read Models

Read model - структура данных, удобная для чтения.

Примеры:

- таблица `order_list_view` для списка заказов;
- денормализованная таблица dashboard metrics;
- Elasticsearch index для поиска;
- Redis cache для counters.

Read model может быть eventually consistent, если обновляется асинхронно из событий.

### Eventual Consistency in CQRS

После записи read model может обновиться не мгновенно.

На интервью важно объяснить UX implications:

- пользователь может увидеть старый статус несколько секунд;
- UI может показывать `processing`;
- нужны retries и monitoring lag;
- критичные commands должны проверять write model, а не stale read model.

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

### Events vs Messages vs Commands

Message - общий контейнер передачи данных.

Event - тип сообщения, описывающий факт прошлого.

Command message - просьба выполнить действие.

```text
Command: ShipOrder
Event: OrderShipped
Message: transport envelope with payload, headers, correlation_id, causation_id
```

Senior-ответ:

> Event не должен знать, кто его обработает. Если отправитель ожидает конкретное действие от конкретного получателя, это скорее command, а не event.

### Brokers

Типичные брокеры:

- RabbitMQ: routing, queues, acknowledgements, retry/dead-letter, хорош для task/event messaging.
- Kafka: distributed log, partitions, consumer groups, replay, высокая throughput, event streaming.
- Redis Streams: проще, но меньше guarantees и tooling по сравнению с Kafka.
- AWS SNS/SQS/EventBridge: managed messaging/event routing.

Что важно обсудить:

- delivery semantics: at-most-once, at-least-once, effectively-once;
- ordering: глобальный порядок почти никогда не нужен, порядок обычно в рамках aggregate id/partition key;
- retries и dead-letter queues;
- schema evolution;
- observability: correlation id, tracing, lag, failed messages.

### Reliability Patterns in EDA

Подробности реализации см. в `05-concurrency-reliability.md`.

Коротко:

- Outbox нужен, чтобы атомарно сохранить state и событие в одной DB transaction.
- Inbox нужен, чтобы consumer не обработал duplicate message дважды.
- Idempotency нужна, потому что at-least-once delivery означает возможные дубли.
- Saga/process manager нужен для долгих процессов между aggregates/services.

### Event-Driven Pitfalls

- Использовать events как скрытые synchronous function calls.
- Слишком мелкие события без бизнес-смысла: `FieldChanged` вместо `InvoicePaid`.
- Отсутствие idempotency.
- Нет DLQ, retry policy и monitoring.
- Нет версионирования event schema.
- Event payload содержит внутреннюю ORM-модель или нестабильный JSON.
- Сильная связность через shared database.
- Непонимание ordering guarantees брокера.

## Event Sourcing

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
- replay и построение новых projections;
- хорошо подходит для финансовых, workflow и compliance-доменов;
- естественная связь с CQRS.

Минусы:

- сложность schema evolution;
- сложность debugging для команды без опыта;
- eventual consistency почти неизбежна;
- нужны snapshots для больших streams;
- нельзя легко изменить прошлое событие.

Event Sourcing похож на бухгалтерскую книгу: баланс не хранится напрямую, а вычисляется из операций. Если где-то ошибка, можно пересчитать или добавить компенсирующее событие.

PHP-библиотеки:

- EventSauce;
- Prooph/Event Store;
- Broadway;
- Spatie Laravel Event Sourcing;
- Ecotone.

## Modular Monolith vs Microservices

### Modular Monolith

Modular monolith - один deployable artifact, но код разделен на модули с явными границами.

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
- риск превратиться в Big Ball of Mud;
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

## Практические объяснения за 30 секунд

### Clean Architecture

Clean Architecture отделяет бизнес-правила от деталей. Внутри находятся domain entities и use cases, снаружи - controllers, ORM, queues, framework. Зависимости направлены внутрь. Это повышает тестируемость и делает систему устойчивее к изменению инфраструктуры, но добавляет boilerplate и не всегда нужно для простого CRUD.

### DDD

DDD помогает моделировать сложную предметную область через общий язык, bounded contexts, aggregates, value objects и domain events. Цель - не паттерны ради паттернов, а код, который выражает бизнес-правила и защищает инварианты.

### CQRS

CQRS разделяет write side и read side. Commands меняют состояние и защищают инварианты, queries читают optimized read models. Это полезно при сложных сценариях записи, тяжелых read views и разных требованиях к производительности, но добавляет eventual consistency и operational complexity.

### Event-Driven

Event-Driven Architecture связывает компоненты через события о произошедших фактах. Это снижает coupling и помогает масштабировать реакции на бизнес-события, но требует idempotency, retries, DLQ, schema evolution, observability и понимания delivery guarantees.

### Modular Monolith

Modular monolith дает модульные границы и DDD-подход без сетевой сложности микросервисов. Это часто лучший старт для продукта. Микросервисы стоит выделять позже, когда границы стабильны и есть реальные причины для независимого scaling/deployment.

## Refactoring Examples

### Fat Controller to Use Case

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

### Primitive Obsession to Value Object

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

### Query via Aggregate to Read Model

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

## Common Senior-Level Trade-Offs

### Когда не применять Clean Architecture

- простой CRUD без сложных правил;
- короткоживущий internal tool;
- маленькая команда без архитектурной дисциплины;
- MVP, где важнее проверить гипотезу.

Можно применить частично: выделить use cases для сложных сценариев и оставить простые CRUD на framework conventions.

### Когда DDD оправдан

- сложные бизнес-правила;
- много исключений и инвариантов;
- разные отделы используют разные смыслы одних терминов;
- частые изменения правил;
- высокая цена ошибки в домене.

### Когда CQRS оправдан

- write и read models сильно отличаются;
- сложные read projections;
- высокие нагрузки на чтение;
- нужны async projections;
- read side должен быть денормализован.

### Когда Event-Driven оправдан

- много независимых реакций на бизнес-события;
- интеграции с другими системами;
- нужен decoupling между bounded contexts;
- долгие процессы;
- аудит и replay важны.

### Когда микросервисы оправданы

- независимые команды;
- независимый deploy;
- разные scaling profiles;
- fault isolation;
- bounded contexts уже понятны.

## Architecture Review Checklist

### Boundaries and Dependencies

- Бизнес-логика не живет в controllers/listeners/jobs?
- Domain не зависит от Laravel/Symfony/Doctrine/Eloquent?
- Use cases выражают бизнес-сценарии, а не технические операции?
- Infrastructure реализует ports, а не протекает внутрь?
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

## Дополнительное чтение

- Martin Fowler, Patterns of Enterprise Application Architecture: <https://martinfowler.com/books/eaa.html>
- Martin Fowler, CQRS: <https://martinfowler.com/bliki/CQRS.html>
- Martin Fowler, Event Sourcing: <https://martinfowler.com/eaaDev/EventSourcing.html>
- Martin Fowler, Strangler Fig Application: <https://martinfowler.com/bliki/StranglerFigApplication.html>
- Martin Fowler, Bounded Context: <https://martinfowler.com/bliki/BoundedContext.html>
- Robert C. Martin, Clean Architecture article: <https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html>
- Robert C. Martin, Clean Architecture book: <https://www.oreilly.com/library/view/clean-architecture-a/9780134494272/>
- Alistair Cockburn, Hexagonal Architecture: <https://alistair.cockburn.us/hexagonal-architecture/>
- Eric Evans, Domain-Driven Design: <https://www.domainlanguage.com/ddd/>
- Vaughn Vernon, Implementing Domain-Driven Design: <https://vaughnvernon.co/?page_id=168>
- Vaughn Vernon, Domain-Driven Design Distilled: <https://vaughnvernon.co/?page_id=168>
- Designing Data-Intensive Applications, Martin Kleppmann: <https://dataintensive.net/>
- Microsoft, CQRS pattern: <https://learn.microsoft.com/en-us/azure/architecture/patterns/cqrs>
- Microsoft, Event Sourcing pattern: <https://learn.microsoft.com/en-us/azure/architecture/patterns/event-sourcing>
- Microsoft, Saga distributed transactions pattern: <https://learn.microsoft.com/en-us/azure/architecture/reference-architectures/saga/saga>
- AWS Prescriptive Guidance, Strangler Fig pattern: <https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/strangler-fig.html>
- AWS Prescriptive Guidance, Transactional outbox pattern: <https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/transactional-outbox.html>
- Symfony Messenger: <https://symfony.com/doc/current/messenger.html>
- Symfony EventDispatcher: <https://symfony.com/doc/current/components/event_dispatcher.html>
- Laravel Events: <https://laravel.com/docs/events>
- Laravel Queues: <https://laravel.com/docs/queues>
- EventSauce: <https://eventsauce.io/>
- Prooph Event Store: <https://github.com/prooph/event-store>
- Broadway: <https://github.com/broadway/broadway>
- Spatie Laravel Event Sourcing: <https://spatie.be/docs/laravel-event-sourcing>
- Ecotone: <https://docs.ecotone.tech/>
