# Architecture Interview Conspectuses

<!-- markdownlint-disable MD013 -->

Набор конспектов для подготовки к senior/lead backend interviews. Файлы собраны на основе старых заметок из `job-interviews/` и перегруппированы по уровню абстракции.

## Как читать

1. `01-design-principles.md` - базовые принципы проектирования кода: SOLID, KISS, DRY, GRASP, coupling/cohesion.
2. `02-design-patterns-php.md` - GoF, PHP/Laravel/Symfony patterns, ORM-паттерны, тестируемость, производительность и антипаттерны.
3. `03-clean-ddd-cqrs-event-driven.md` - Clean/Hexagonal Architecture, DDD, CQRS, Event Sourcing, Event-Driven Architecture, modular monolith и microservices.
4. `04-system-design-interview.md` - как проходить system design interview: requirements, capacity, API, storage, caching, queues, consistency, operations.
5. `05-concurrency-reliability.md` - concurrency, locks, idempotency, delivery guarantees, outbox/inbox, sagas, retries, DLQ, backpressure.
6. `99-coverage-map.md` - карта покрытия старых файлов и новых конспектов.

## Границы файлов

Главное правило новой структуры: одна тема может упоминаться в нескольких местах, но глубокое объяснение находится только в одном файле.

| Тема | Основной файл | В остальных файлах |
|---|---|---|
| SOLID, KISS, DRY, YAGNI | `01-design-principles.md` | краткие ссылки и применение |
| GoF, GRASP, PHP framework patterns | `02-design-patterns-php.md` | только примеры использования |
| Clean/Hexagonal, DDD, CQRS, EDA | `03-clean-ddd-cqrs-event-driven.md` | только system design или reliability implications |
| System design interview flow | `04-system-design-interview.md` | не дублируется |
| Locks, idempotency, outbox, inbox, sagas | `05-concurrency-reliability.md` | кратко в architecture/system design |

## Базовые источники

- Martin Fowler, Architecture: <https://martinfowler.com/architecture/>
- Martin Fowler, Patterns of Enterprise Application Architecture: <https://martinfowler.com/books/eaa.html>
- Robert C. Martin, Clean Architecture article: <https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html>
- Eric Evans, Domain-Driven Design: <https://www.domainlanguage.com/ddd/>
- Martin Kleppmann, Designing Data-Intensive Applications: <https://dataintensive.net/>
- System Design Primer: <https://github.com/donnemartin/system-design-primer>
- Refactoring Guru, Design Patterns in PHP: <https://refactoring.guru/design-patterns/php>
