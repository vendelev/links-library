# Design Patterns in PHP 8, Laravel and Symfony

<!-- markdownlint-disable MD013 -->

Конспект по классическим и практическим паттернам в PHP backend. Фокус: GoF, GRASP, Laravel/Symfony специфика, ORM-паттерны, тестируемость, производительность и антипаттерны.

## Как говорить о паттернах

Паттерн - это шаблон решения повторяющейся архитектурной задачи, а не обязательная конструкция. На интервью важно объяснить:

- какую проблему решает паттерн;
- какой trade-off добавляет;
- как он проявляется в PHP/Laravel/Symfony;
- когда он становится overengineering;
- как влияет на тестируемость, coupling и performance.

## Framework Terms vs GoF Patterns

### Facade

GoF Facade предоставляет упрощенный интерфейс к сложной подсистеме.

Laravel Facade - статический proxy к объекту из service container. Например, `Cache::get()` перенаправляется к сервису кеша. Технически Laravel Facade ближе к Proxy/Service Locator, хотя внешне сохраняет идею упрощенного интерфейса.

Symfony обычно предпочитает явный Dependency Injection и сервисы.

### Factory

Factory Method и Abstract Factory управляют созданием объектов и скрывают детали инстанцирования.

Laravel Model Factory - инструмент для генерации тестовых данных и seed data. Это не то же самое, что GoF Factory.

Symfony service factories в container configuration ближе к классическому Factory: контейнер вызывает указанный метод, чтобы сконструировать объект сервиса.

### Active Record vs Data Mapper

Laravel Eloquent реализует Active Record: модель содержит данные и методы persistence.

Doctrine ORM использует Data Mapper: entity не знает о БД, а `EntityManager` и Unit of Work занимаются загрузкой и сохранением.

Trade-off:

- Active Record проще и быстрее для CRUD и небольших проектов;
- Data Mapper лучше изолирует domain model от persistence в сложном домене;
- Data Mapper дороже в настройке и требует больше mapping;
- Active Record может привести к leakage БД в бизнес-логику.

## Creational Patterns

### Factory Method

Factory Method определяет интерфейс создания объекта, но позволяет подклассам или конфигурации решать, какой класс создать.

Пример: создание `FileLogger` или `DbLogger` в зависимости от настроек.

### Abstract Factory

Abstract Factory создает семейство связанных объектов без привязки к конкретным классам.

Пример: набор адаптеров для разных payment providers или набор UI components для разных тем.

### Builder

Builder поэтапно конструирует сложный объект.

Примеры в PHP:

- Laravel Query Builder;
- Doctrine QueryBuilder;
- сборка OpenAPI schema;
- генерация сложного отчета.

### Prototype

Prototype создает объект клонированием существующего. В PHP используется через `clone`.

Подходит, если создание объекта дорого, а нужно получить похожий экземпляр с небольшими изменениями.

### Singleton

Singleton гарантирует один экземпляр и глобальную точку доступа.

Минусы:

- скрытая глобальность;
- сложнее тестирование;
- состояние течет между тестами;
- tight coupling к способу доступа.

В большинстве backend-сценариев лучше DI container с shared service, чем Singleton в доменной логике.

## Structural Patterns

### Adapter

Adapter приводит интерфейс одного класса к виду, ожидаемому клиентом.

Примеры:

- единый `PaymentGateway` поверх Stripe, PayPal и Adyen;
- `CacheInterface` поверх Redis, Memcached, filesystem;
- Flysystem для local/S3/FTP storage.

### Decorator

Decorator динамически добавляет поведение объекту, сохраняя интерфейс.

Примеры:

- кеширующий decorator над repository;
- logging decorator над HTTP client;
- retry decorator над payment gateway;
- Symfony service decoration.

### Facade

Facade дает простой интерфейс к подсистеме.

Пример: `CheckoutFacade` может координировать order, payment, inventory и notification modules, но важно не превратить его в God Object.

### Proxy

Proxy контролирует доступ к объекту.

Примеры:

- lazy proxy в Doctrine associations;
- lazy services в Symfony;
- Laravel Facade как static proxy;
- remote proxy для внешнего API.

### Composite

Composite организует объекты в дерево и позволяет работать с листами и группами единообразно.

Примеры:

- HTML tree;
- меню с nested items;
- permission tree;
- AST.

### Bridge

Bridge разделяет абстракцию и реализацию.

Пример: `ReportRenderer` использует `PdfEngine` или `HtmlEngine`, которые можно менять независимо.

### Flyweight

Flyweight разделяет общее состояние между множеством мелких объектов.

Пример: reuse immutable value objects, metadata objects, parsed configuration.

## Behavioral Patterns

### Strategy

Strategy инкапсулирует семейство взаимозаменяемых алгоритмов.

Пример без паттерна:

```php
final class CheckoutController
{
    public function pay(Order $order, string $method): void
    {
        if ($method === 'card') {
            // card payment
        } elseif ($method === 'paypal') {
            // paypal payment
        }
    }
}
```

Вариант через Strategy:

```php
interface PaymentStrategy
{
    public function pay(Order $order): PaymentResult;
}

final class CardPayment implements PaymentStrategy
{
    public function pay(Order $order): PaymentResult
    {
        // Card provider call.
    }
}
```

Плюсы: расширяемость, тестируемость, изоляция алгоритмов. Минусы: больше классов, overkill для одноразовой ветки.

### Observer

Observer устанавливает связь publisher-subscriber.

Пример: после `UserRegistered` отправить welcome email, создать запись в CRM и уведомить админа.

Laravel Events и Symfony EventDispatcher реализуют этот стиль.

Плюсы: low coupling, легко добавить реакцию. Минусы: flow становится неочевидным, сложнее транзакционные гарантии, нужны правила ошибок и retries.

### Command

Command инкапсулирует запрос в объект.

Примеры:

- Symfony Messenger message;
- Laravel Job;
- CQRS command;
- CLI command;
- undo/redo operation.

Важно: command выражает намерение выполнить действие, event фиксирует факт прошлого.

### Chain of Responsibility

Запрос проходит по цепочке handlers.

Примеры:

- HTTP middleware;
- validation pipeline;
- Monolog handlers;
- Guzzle middleware stack.

### Template Method

Базовый класс задает скелет алгоритма, подклассы реализуют отдельные шаги.

Пример: общий export flow, где `CsvExporter` и `PdfExporter` меняют только генерацию содержимого.

Минус: наследование может нарушить LSP или привести к rigid hierarchy. Часто Strategy или composition безопаснее.

### State

State меняет поведение объекта в зависимости от состояния.

Пример: заказ в состояниях `new`, `paid`, `shipped`, `cancelled`; методы `pay()`, `ship()`, `cancel()` ведут себя по-разному.

Для простых lifecycle достаточно enum + guard methods. State pattern оправдан, когда переходов и поведения много.

### Mediator

Mediator централизует взаимодействие множества объектов.

Примеры:

- message bus;
- command bus;
- chat room;
- UI dialog coordinator.

Минус: mediator может стать God Object, если туда складывают бизнес-логику.

### Memento

Memento сохраняет snapshot состояния без нарушения инкапсуляции.

Примеры: undo, draft state, restore point. В backend встречается редко, но идея близка к snapshots в Event Sourcing.

### Visitor

Visitor добавляет операции над структурой объектов без изменения самих классов.

Примеры: обход AST, экспорт разных document nodes, static analysis tools.

### Iterator

Iterator дает последовательный доступ к коллекции без раскрытия внутренней структуры.

В PHP часто неявно используется через `Iterator`, `IteratorAggregate`, generators и `yield`.

### Interpreter

Interpreter описывает грамматику простого языка и интерпретирует выражения.

Примеры: DSL, formulas, rules engine, template language. В PHP приложениях чаще используют готовые parser libraries.

## GRASP as Pattern Selection Guide

GRASP помогает решить, кому дать ответственность, до выбора GoF-паттерна.

Краткая шпаргалка:

- Information Expert: логика у объекта, владеющего данными.
- Creator: создает тот, кто содержит или тесно использует объект.
- Controller: системное событие принимает use case/controller.
- Low Coupling: меньше знания о соседях.
- High Cohesion: связанные обязанности вместе.
- Polymorphism: не `switch` по типу, а разные реализации.
- Protected Variations: защитить изменчивые места интерфейсами/adapters.
- Indirection: посредник разрывает прямую связь.
- Pure Fabrication: искусственный сервис допустим, если улучшает design.

## Patterns and Testability

Паттерны улучшают тестируемость, когда уменьшают coupling и делают зависимости явными.

Хорошо помогают:

- Dependency Injection: можно подменить dependency mock/fake;
- Strategy: можно подставить тестовый алгоритм;
- Command Handler: можно тестировать use case отдельно;
- Repository interface: можно использовать in-memory fake;
- Observer/Event Dispatcher: можно заменить fake dispatcher или `Event::fake()`;
- Factory: можно создавать предсказуемые объекты в тестах.

Ухудшают тестируемость:

- Singleton;
- Service Locator;
- глобальное состояние;
- Fat Controller или God Service;
- скрытые side effects;
- статические вызовы без возможности подмены.

Laravel Facades статичны, но framework дает механизмы mocking/faking. Это снижает боль, но не отменяет необходимость понимать скрытые зависимости.

## Patterns and Performance

Паттерны добавляют абстракции и иногда дополнительные вызовы, но в backend чаще bottleneck находится в БД, сети, файловой системе или внешних API.

Что важно:

- не создавать лишние объекты в горячих циклах без профилирования;
- помнить про N+1 при ORM lazy loading;
- использовать eager loading там, где access pattern известен;
- кеширующий decorator может улучшить latency;
- lazy proxy экономит ресурсы, но может скрыть дополнительные queries;
- CQRS может ускорить чтение через read models, но добавляет eventual consistency.

Пример CQRS performance trade-off: если 90 процентов нагрузки - чтение каталога, read side можно вынести на replicas, Elasticsearch или денормализованные таблицы. Цена - синхронизация и lag.

## Anti-Patterns

### God Object

Класс делает все: регистрация, пароль, email, отчеты, persistence. Решение: декомпозиция по responsibilities, SRP, application services, domain services, adapters.

### Spaghetti Code

Логика переплетена без структуры. Решение: слои, use cases, маленькие функции, тесты перед рефакторингом.

### Singleton Abuse

Глобальная точка доступа используется вместо явных зависимостей. Решение: DI container, explicit dependencies, scoped shared services.

### Service Locator

Класс сам лезет в контейнер за зависимостями. Это скрывает контракт класса. Решение: constructor injection.

### Anemic Domain Model

Entity содержит только getters/setters, вся логика в services. Риск: инварианты размазаны и дублируются. Решение: rich domain model, методы поведения на агрегатах.

### Fat Controller

Controller содержит orchestration, SQL, валидацию, внешние вызовы и business rules. Решение: thin controller, application use case, domain model.

### Fat Model

В Active Record модель может стать God Object: ORM, business rules, validation, events, integration. Решение: value objects, domain services, repositories/adapters, read/write separation при необходимости.

### Golden Hammer

Любимый паттерн применяется ко всем задачам. Решение: KISS, YAGNI и выбор инструмента под проблему.

### Heavy Constructor

Конструктор делает дорогие загрузки или тянет полприложения. Решение: lazy loading, factory, builder, explicit initialization.

### Copy-Paste Programming

Повтор кода вместо выделения общего знания. Но перед DRY нужно проверить, совпадает ли причина изменения.

### Early Optimization

Оптимизация без профилирования вредит читаемости. Решение: измерять, потом оптимизировать bottleneck.

## PHP Ecosystem Examples

- Laravel: Eloquent Active Record, Facades as Proxy/Service Locator, Query Builder, Events, Middleware, Queue jobs, cache drivers as Strategy.
- Symfony: HttpKernel Front Controller, Service Container, EventDispatcher, Messenger, service decoration, Lock component.
- Doctrine: Data Mapper, Unit of Work, Identity Map, Repository, Lazy Proxy, QueryBuilder.
- PSR-3 Logger: Strategy/Adapter style logging interface.
- PSR-6/PSR-16 Cache: cache abstraction over different backends.
- PSR-11 Container: container interface, can become Service Locator if used inside business code.
- PSR-14 Event Dispatcher: Observer.
- PSR-15 Middleware: Chain of Responsibility.
- Monolog: Chain of Responsibility for handlers and decorators/formatters.
- Flysystem: Adapter over filesystems.
- Tactician/SimpleBus/Symfony Messenger: Command Bus.
- Broadway/EventSauce/Spatie Event Sourcing/Ecotone: CQRS and Event Sourcing tooling.
- Guzzle: Client Facade and middleware stack.
- PHPUnit: Template Method (`setUp`, `tearDown`), Composite suites, Observer listeners.

## Analogies for Interviews

- Facade: metrdotel in restaurant hides kitchen complexity.
- Strategy: choose car, bike or walk depending on conditions.
- Observer: YouTube subscribers receive new video notification.
- Decorator: Christmas tree with garland and toys.
- Repository: librarian retrieves books without exposing storage details.
- Unit of Work: secretary collects changes and applies them together.
- CQRS: warehouse writes authoritative stock, shop window reads optimized view.
- Event Sourcing: bank account balance as sum of transaction history.
- DI: device gets electricity from socket instead of producing it itself.
- LSP: one TV remote works with different TVs through the same interface.

## Pattern Cheat Sheet

| Group | Patterns | Main Use |
|---|---|---|
| Creational | Factory Method, Abstract Factory, Builder, Prototype, Singleton | object creation |
| Structural | Adapter, Bridge, Composite, Decorator, Facade, Flyweight, Proxy | object relationships |
| Behavioral | Chain, Command, Iterator, Mediator, Memento, Observer, State, Strategy, Template Method, Visitor | object collaboration |
| Enterprise | Repository, Unit of Work, Identity Map, Data Mapper, Active Record | persistence and application architecture |
| Architecture | Clean, Hexagonal, DDD, CQRS, Event Sourcing | high-level system design |

## Self-Check Questions

1. Чем Laravel Facade отличается от GoF Facade?
2. Когда Strategy лучше `match`, а когда хуже?
3. Почему Singleton усложняет тесты?
4. Чем Active Record отличается от Data Mapper?
5. Как Observer снижает coupling и почему ухудшает traceability?
6. Чем Command отличается от Event?
7. Где Template Method хуже composition?
8. Почему Service Locator скрывает зависимости?
9. Как Decorator помогает добавить caching/retry/logging?
10. Какие паттерны чаще всего встречаются в Symfony Messenger и Laravel Queue?

## Дополнительное чтение

- Refactoring Guru, Design Patterns in PHP: <https://refactoring.guru/design-patterns/php>
- DesignPatternsPHP examples: <https://designpatternsphp.readthedocs.io/>
- Martin Fowler, Patterns of Enterprise Application Architecture catalog: <https://martinfowler.com/eaaCatalog/>
- Martin Fowler, Dependency Injection: <https://martinfowler.com/articles/injection.html>
- PHP Manual, Classes and Objects: <https://www.php.net/manual/en/language.oop5.php>
- PHP-FIG PSR-3 Logger Interface: <https://www.php-fig.org/psr/psr-3/>
- PHP-FIG PSR-6 Cache Interface: <https://www.php-fig.org/psr/psr-6/>
- PHP-FIG PSR-11 Container Interface: <https://www.php-fig.org/psr/psr-11/>
- PHP-FIG PSR-14 Event Dispatcher: <https://www.php-fig.org/psr/psr-14/>
- PHP-FIG PSR-15 HTTP Server Handlers: <https://www.php-fig.org/psr/psr-15/>
- Laravel Facades: <https://laravel.com/docs/facades>
- Laravel Eloquent Factories: <https://laravel.com/docs/eloquent-factories>
- Laravel Service Container: <https://laravel.com/docs/container>
- Laravel Events: <https://laravel.com/docs/events>
- Symfony Service Container: <https://symfony.com/doc/current/service_container.html>
- Symfony EventDispatcher: <https://symfony.com/doc/current/components/event_dispatcher.html>
- Symfony Messenger: <https://symfony.com/doc/current/messenger.html>
- Doctrine ORM Architecture: <https://www.doctrine-project.org/projects/doctrine-orm/en/current/reference/architecture.html>
- Doctrine Unit of Work: <https://www.doctrine-project.org/projects/doctrine-orm/en/current/reference/unitofwork.html>
- Doctrine Associations: <https://www.doctrine-project.org/projects/doctrine-orm/en/current/reference/working-with-associations.html>
- EventSauce: <https://eventsauce.io/>
- Spatie Laravel Event Sourcing: <https://spatie.be/docs/laravel-event-sourcing>
- Ecotone: <https://docs.ecotone.tech/>
