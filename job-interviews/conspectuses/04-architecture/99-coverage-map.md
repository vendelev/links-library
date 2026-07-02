# Coverage Map

<!-- markdownlint-disable MD013 -->

Карта проверки переноса информации из исходных конспектов в новые файлы `job-interviews/conspectuses/04-architecture`.

## Исходные файлы

1. `job-interviews/SOLID KISS DRY и принципы проектирования.md`
2. `job-interviews/Паттерны_программирования_в_PHP_8+_подробный_гайд_для_Senior_Backend.md`
3. `job-interviews/Clean Architecture DDD CQRS Event-Driven.md`
4. `job-interviews/System Design для Lead Senior Backend.md`
5. `job-interviews/Concurrency distributed systems locks idempotency outbox sagas.md`

## Новые файлы

1. `00-index.md`
2. `01-design-principles.md`
3. `02-design-patterns-php.md`
4. `03-clean-ddd-cqrs-event-driven.md`
5. `04-system-design-interview.md`
6. `05-concurrency-reliability.md`

## Что куда перенесено

| Тема | Старые файлы | Новый основной файл | Статус |
|---|---|---|---|
| Как отвечать на интервью по принципам | SOLID | `01-design-principles.md` | перенесено |
| SOLID: SRP, OCP, LSP, ISP, DIP | SOLID, Patterns | `01-design-principles.md` | перенесено, в Patterns оставлено через references |
| KISS, DRY, YAGNI | SOLID, Patterns | `01-design-principles.md` | перенесено |
| DRY vs premature abstraction | SOLID, Patterns | `01-design-principles.md` | перенесено |
| OCP vs overengineering | SOLID, Patterns | `01-design-principles.md` | перенесено |
| DIP vs interface for everything | SOLID, Clean Architecture | `01-design-principles.md` | перенесено |
| GRASP | SOLID, Patterns | `01-design-principles.md`, `02-design-patterns-php.md` | перенесено |
| Law of Demeter | SOLID | `01-design-principles.md` | перенесено |
| Composition over inheritance | SOLID | `01-design-principles.md` | перенесено |
| High cohesion / low coupling | SOLID, Patterns | `01-design-principles.md` | перенесено |
| Tell Don't Ask | SOLID | `01-design-principles.md` | перенесено |
| Fail Fast | SOLID | `01-design-principles.md` | перенесено |
| Separation of Concerns | SOLID, Patterns, Clean Architecture | `01-design-principles.md`, `03-clean-ddd-cqrs-event-driven.md` | перенесено |
| Code review checklist по дизайну | SOLID | `01-design-principles.md` | перенесено |
| Laravel Facade vs GoF Facade | Patterns | `02-design-patterns-php.md` | перенесено |
| Laravel Factory vs GoF Factory | Patterns | `02-design-patterns-php.md` | перенесено |
| Active Record vs Data Mapper | Patterns, Clean Architecture | `02-design-patterns-php.md`, `03-clean-ddd-cqrs-event-driven.md` | перенесено |
| GoF Creational patterns | Patterns | `02-design-patterns-php.md` | перенесено |
| GoF Structural patterns | Patterns | `02-design-patterns-php.md` | перенесено |
| GoF Behavioral patterns | Patterns | `02-design-patterns-php.md` | перенесено |
| Strategy example | SOLID, Patterns | `02-design-patterns-php.md` | перенесено |
| Observer example | Patterns, Clean Architecture | `02-design-patterns-php.md`, `03-clean-ddd-cqrs-event-driven.md` | перенесено |
| Command pattern | Patterns, Clean Architecture | `02-design-patterns-php.md`, `03-clean-ddd-cqrs-event-driven.md` | перенесено |
| Chain of Responsibility / Middleware | Patterns, System Design | `02-design-patterns-php.md` | перенесено |
| Patterns and testability | Patterns, SOLID | `02-design-patterns-php.md` | перенесено |
| Patterns and performance | Patterns, System Design | `02-design-patterns-php.md`, `04-system-design-interview.md` | перенесено |
| Anti-patterns: God Object, Spaghetti, Singleton, Service Locator, Anemic Model, Fat Controller, Fat Model, Golden Hammer | Patterns, SOLID, Clean Architecture | `02-design-patterns-php.md`, `03-clean-ddd-cqrs-event-driven.md` | перенесено |
| PHP ecosystem libraries and PSR patterns | Patterns | `02-design-patterns-php.md` | перенесено |
| Analogies for patterns | Patterns | `02-design-patterns-php.md` | перенесено |
| Pattern cheat sheet | Patterns | `02-design-patterns-php.md` | перенесено |
| Clean Architecture layers | Clean Architecture | `03-clean-ddd-cqrs-event-driven.md` | перенесено |
| Dependency Rule | Clean Architecture, SOLID | `03-clean-ddd-cqrs-event-driven.md` | перенесено |
| Entities and Use Cases | Clean Architecture | `03-clean-ddd-cqrs-event-driven.md` | перенесено |
| Interface Adapters | Clean Architecture | `03-clean-ddd-cqrs-event-driven.md` | перенесено |
| Frameworks and Drivers as details | Clean Architecture | `03-clean-ddd-cqrs-event-driven.md` | перенесено |
| Clean Architecture pitfalls and trade-offs | Clean Architecture | `03-clean-ddd-cqrs-event-driven.md` | перенесено |
| Hexagonal Architecture / Ports and Adapters | Clean Architecture, SOLID | `03-clean-ddd-cqrs-event-driven.md` | перенесено |
| DDD: Ubiquitous Language | Clean Architecture, Patterns | `03-clean-ddd-cqrs-event-driven.md` | перенесено |
| DDD: Entity, Value Object, Aggregate, Aggregate Root, Repository, Domain Service, Application Service, Domain Event, Bounded Context | Clean Architecture, Patterns | `03-clean-ddd-cqrs-event-driven.md` | перенесено |
| DDD pitfalls | Clean Architecture, Patterns | `03-clean-ddd-cqrs-event-driven.md` | перенесено |
| CQRS: commands, queries, read models, eventual consistency | Clean Architecture, Patterns, System Design | `03-clean-ddd-cqrs-event-driven.md` | перенесено |
| CQRS in Symfony/Laravel | Clean Architecture, Patterns | `03-clean-ddd-cqrs-event-driven.md` | перенесено |
| Event-Driven Architecture | Clean Architecture, Patterns, System Design, Concurrency | `03-clean-ddd-cqrs-event-driven.md` | перенесено |
| Events vs messages vs commands | Clean Architecture, Patterns | `03-clean-ddd-cqrs-event-driven.md` | перенесено |
| Brokers: RabbitMQ, Kafka, Redis Streams, SNS/SQS/EventBridge | Clean Architecture, System Design, Concurrency | `03-clean-ddd-cqrs-event-driven.md`, `04-system-design-interview.md`, `05-concurrency-reliability.md` | перенесено |
| Event Sourcing | Clean Architecture, Patterns | `03-clean-ddd-cqrs-event-driven.md` | перенесено |
| Modular Monolith vs Microservices | Clean Architecture, System Design | `03-clean-ddd-cqrs-event-driven.md` | перенесено |
| Strangler Fig | Clean Architecture | `03-clean-ddd-cqrs-event-driven.md` | перенесено |
| Architecture review checklist | Clean Architecture | `03-clean-ddd-cqrs-event-driven.md` | перенесено |
| System design interview flow | System Design | `04-system-design-interview.md` | перенесено |
| Requirements clarification | System Design | `04-system-design-interview.md` | перенесено |
| Functional and non-functional requirements | System Design | `04-system-design-interview.md` | перенесено |
| Capacity estimates | System Design | `04-system-design-interview.md` | перенесено |
| High-level architecture for PHP backend | System Design | `04-system-design-interview.md` | перенесено |
| API design, errors, idempotency header | System Design | `04-system-design-interview.md` | перенесено |
| Data model and indexes | System Design | `04-system-design-interview.md` | перенесено |
| Storage choice: PostgreSQL, Redis, ClickHouse, RabbitMQ/SQS, Object Storage | System Design | `04-system-design-interview.md` | перенесено |
| Caching strategies and problems | System Design | `04-system-design-interview.md` | перенесено |
| Queues and async processing | System Design, Concurrency | `04-system-design-interview.md`, `05-concurrency-reliability.md` | перенесено |
| Consistency in system design | System Design, Clean Architecture, Concurrency | `04-system-design-interview.md`, `05-concurrency-reliability.md` | перенесено |
| Availability | System Design | `04-system-design-interview.md` | перенесено |
| Scalability | System Design | `04-system-design-interview.md` | перенесено |
| Partitioning and sharding | System Design | `04-system-design-interview.md` | перенесено |
| Replication and lag | System Design, Concurrency | `04-system-design-interview.md`, `05-concurrency-reliability.md` | перенесено |
| Load balancing | System Design | `04-system-design-interview.md` | перенесено |
| Rate limiting | System Design, Concurrency | `04-system-design-interview.md`, `05-concurrency-reliability.md` | перенесено |
| Retries and circuit breakers | System Design, Concurrency | `04-system-design-interview.md`, `05-concurrency-reliability.md` | перенесено |
| Observability | System Design, Clean Architecture, Concurrency | `04-system-design-interview.md`, `05-concurrency-reliability.md` | перенесено |
| Security basics | System Design | `04-system-design-interview.md` | перенесено |
| Deployment and rollout | System Design | `04-system-design-interview.md` | перенесено |
| Disaster recovery | System Design | `04-system-design-interview.md` | перенесено |
| Common interview tasks: URL shortener, Feed, Marketplace Analytics, Notification Service, File/Media Service, Order Processing | System Design | `04-system-design-interview.md` | перенесено |
| Concurrency and parallelism | Concurrency | `05-concurrency-reliability.md` | перенесено |
| Race condition and critical section | Concurrency | `05-concurrency-reliability.md` | перенесено |
| Lost update, check-then-act, double submit, duplicate message processing | Concurrency | `05-concurrency-reliability.md` | перенесено |
| Optimistic locking | Concurrency, System Design | `05-concurrency-reliability.md` | перенесено |
| Pessimistic locking | Concurrency | `05-concurrency-reliability.md` | перенесено |
| PostgreSQL row locks, NOWAIT, SKIP LOCKED, advisory locks, isolation levels | Concurrency | `05-concurrency-reliability.md` | перенесено |
| Atomic operations and constraints | Concurrency, System Design | `05-concurrency-reliability.md` | перенесено |
| Distributed locks, Redis lock, Laravel atomic locks, Symfony Lock, Redlock caveats, fencing tokens | Concurrency | `05-concurrency-reliability.md` | перенесено |
| Idempotency key storage and pitfalls | Concurrency, System Design | `05-concurrency-reliability.md` | перенесено |
| Exactly-once myth | Concurrency, System Design | `05-concurrency-reliability.md` | перенесено |
| At-least-once and at-most-once delivery | Concurrency, Clean Architecture, System Design | `05-concurrency-reliability.md` | перенесено |
| Deduplication and inbox table | Concurrency, Clean Architecture | `05-concurrency-reliability.md` | перенесено |
| Outbox table, publisher worker, CDC alternative | Concurrency, Clean Architecture, System Design | `05-concurrency-reliability.md` | перенесено |
| Saga choreography and orchestration/process manager | Concurrency, Clean Architecture, System Design | `05-concurrency-reliability.md` | перенесено |
| Read your writes | Concurrency, System Design, Clean Architecture | `05-concurrency-reliability.md` | перенесено |
| Backpressure | Concurrency | `05-concurrency-reliability.md` | перенесено |
| RabbitMQ ack/nack, DLQ, poison messages | Concurrency, System Design | `05-concurrency-reliability.md` | перенесено |
| Redis SET NX PX and deduplication window | Concurrency | `05-concurrency-reliability.md` | перенесено |
| PHP-FPM, Laravel, Symfony concurrency pitfalls | Concurrency | `05-concurrency-reliability.md` | перенесено |
| Typical senior answers по reliability | Concurrency | `05-concurrency-reliability.md` | перенесено |

## Что было намеренно не продублировано дословно

Новые файлы не являются побайтовым копированием исходников. Повторы были объединены, а темы разнесены по основному месту ответственности.

Сокращены без потери смысла:

- повторные определения DDD/CQRS/Event Sourcing из файла про паттерны, потому что глубокое объяснение находится в `03-clean-ddd-cqrs-event-driven.md`;
- повторные объяснения outbox/inbox/saga из architecture/system design, потому что подробная реализация находится в `05-concurrency-reliability.md`;
- повторные объяснения SOLID/KISS/DRY из файла про паттерны, потому что подробное объяснение находится в `01-design-principles.md`;
- одинаковые interview checklists объединены в профильные checklists;
- ссылки из исходных файлов сохранены и расширены в разделах `Дополнительное чтение`.

## Проверка ссылок на источники

Ссылки из исходников перенесены по темам:

- SOLID, DI, GRASP, Law of Demeter - `01-design-principles.md`.
- GoF, PSR, Laravel/Symfony, Doctrine, EventSauce/Spatie/Ecotone - `02-design-patterns-php.md`.
- Clean Architecture, Hexagonal, DDD, CQRS, Event Sourcing, Saga, Strangler Fig - `03-clean-ddd-cqrs-event-driven.md`.
- AWS, Google SRE, Azure Architecture, Redis, RabbitMQ, PostgreSQL, ClickHouse, SQS, OpenAPI, OWASP - `04-system-design-interview.md`.
- Distributed systems, locks, PostgreSQL locking/isolation/advisory locks, Redis locks, RabbitMQ confirms/prefetch/DLX, Laravel/Symfony lock/queue docs, Debezium outbox - `05-concurrency-reliability.md`.

## Итог проверки покрытия

Информация из исходных файлов перенесена в новую структуру по смыслу. Явных потерянных тем не осталось: все крупные разделы, примеры, pitfalls, checklists, practice tasks и ссылки распределены по новым файлам.
