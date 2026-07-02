# Symfony: internals, DI, HttpKernel, Doctrine, Messenger

Цель: быстро повторить Symfony перед интервью уровня Middle+/Senior/Lead PHP developer.

Фокус: HttpKernel, EventDispatcher, compiled DI container, bundles, Doctrine, transactions,
Messenger и тестирование.

## Быстрый Senior-Ответ

Symfony явно построен вокруг HttpKernel, EventDispatcher и скомпилированного DI container.

Controller выбирается через routing и controller resolver, аргументы собираются value resolvers,
расширяемость часто делается через events/listeners/subscribers, compiler passes и bundles.

В backend-интервью важно уверенно объяснять container compilation, private services,
Doctrine Unit of Work/Identity Map, lazy proxies, transaction boundaries, Messenger retries,
failure transport и integration testing.

## Request Lifecycle

Упрощенный путь HTTP-запроса:

1. Front controller `public/index.php` создает Kernel.
2. Kernel загружает bundles, config и container.
3. HttpKernel обрабатывает `Request` через EventDispatcher.
4. `kernel.request`: routing, locale, security listeners и другие request listeners.
5. Router определяет controller и route attributes.
6. Argument resolver собирает аргументы controller action.
7. Controller возвращает `Response` или данные, преобразуемые listeners.
8. `kernel.response` модифицирует response.
9. `kernel.terminate` выполняет post-response задачи.

Практические детали:

- Symfony container компилируется, оптимизируется и удаляет неиспользуемые private services.
- Controller invocation идет через resolver-ы, а не через простой `new Controller()`.
- Большая часть расширяемости находится в events/listeners/subscribers/compiler passes.
- Многие компоненты Symfony независимы и используются в других фреймворках: Console, HttpFoundation,
  Routing, EventDispatcher, VarDumper.

Короткий ответ: Kernel передает Request в HttpKernel, события и routing выбирают controller.
Value resolvers собирают аргументы, response проходит через response/terminate events.

## DI Container

Что надо знать:

- Services обычно private by default.
- Autowiring подбирает зависимости по type-hint.
- Autoconfiguration добавляет tags/interfaces автоматически: commands, event subscribers, validators и т.д.
- Attributes помогают конфигурировать injection, routes, listeners и autowire aliases.
- Compiler pass позволяет менять container на этапе компиляции.

Типичные ошибки:

- Делать все services public, чтобы доставать их из container.
- Прятать зависимости через service locator без необходимости.
- Игнорировать compile-time ошибки container и пытаться решать их runtime-хаками.
- Не понимать разницу между autowiring alias и concrete service id.

Пример:

```yaml
# config/services.yaml
services:
  App\:
    resource: '../src/'
    exclude: '../src/{DependencyInjection,Entity,Kernel.php}'

  App\Billing\PaymentGatewayInterface:
    alias: App\Billing\StripePaymentGateway
```

Senior-пояснение: Symfony DI стремится к compile-time проверке графа зависимостей.
Это снижает runtime-сюрпризы и делает приложение более предсказуемым.

## Bundles

Bundle - модуль расширения приложения или reusable package.

- `Bundle` class подключает расширение.
- `Extension` загружает config и services.
- Compiler passes модифицируют container до компиляции.
- Recipes/Flex помогают устанавливать config для bundles.

Питфоллы:

- Тащить bundle ради маленькой функции.
- Писать business logic внутри bundle extension.
- Не документировать extension config tree.

## HttpKernel Events

Основные события:

- `kernel.request`: до controller, можно изменить request или вернуть response.
- `kernel.controller`: после выбора controller.
- `kernel.controller_arguments`: после resolution аргументов.
- `kernel.view`: если controller вернул не `Response`.
- `kernel.response`: перед отправкой response.
- `kernel.exception`: обработка исключений.
- `kernel.terminate`: после отправки response.

Senior-пояснение: Symfony чаще мыслится event-driven kernel. Важно не смешивать HTTP concerns с доменной логикой.

## Routing и Controllers

- Routing обычно через attributes, YAML/XML/PHP config.
- Controller arguments resolve-ятся через value resolvers.
- Param converters в новых проектах часто заменяются `MapEntity`/value resolvers.
- Controller должен быть тонким: orchestration, validation, mapping response.

Питфоллы:

- Смешивать entity, form, command и response DTO в одном объекте.
- Перегружать controller security expressions бизнес-правилами.
- Возвращать Doctrine entities напрямую в публичный API без контроля сериализации.

Короткий ответ: Routing listener определяет controller, затем argument/value resolvers собирают аргументы
из request attributes, services, entity mapping, request body и других источников.

## Validation

- Validator component использует constraints через attributes/YAML/XML/PHP.
- Validation groups помогают разным сценариям, но могут усложнить модель.
- Forms полезны для HTML, но для API часто проще DTO + Validator.

Питфоллы:

- Считать constraint доменным правилом, когда правило должно жить в aggregate/service.
- Слишком сложные validation groups вместо явных input DTO.

## Configuration и Env

- Env vars могут использоваться в config через `%env(...)%`.
- Secrets component подходит для encrypted secrets.
- Config normalizer/processor в bundle extension валидирует структуру config.
- Cache warmup важен для production deploy.

Питфоллы:

- Путать build-time и runtime env.
- Делать dynamic config, который ломает container compilation.
- Не различать параметры container и env processors.

## Doctrine ORM Internals

Doctrine ORM - Data Mapper ORM: entity обычно не знает о persistence API, EntityManager управляет состоянием.

Ключевые понятия:

- EntityManager - фасад к Unit of Work, repositories и connection.
- Unit of Work отслеживает изменения entities.
- Identity Map гарантирует один object instance для одной строки в рамках EntityManager.
- `persist()` ставит entity под управление, но не делает INSERT немедленно.
- `flush()` вычисляет changesets и синхронизирует БД.
- Lazy proxies загружают связи при обращении.
- DQL/QueryBuilder работают на уровне object model, DBAL - ближе к SQL.

Питфоллы:

- Огромный Unit of Work в batch jobs без `clear()`.
- Lazy loading после закрытия EntityManager.
- Неочевидные cascades/orphanRemoval.
- Bidirectional relation не синхронизирована с обеих сторон.
- N+1 из-за lazy relations в serializer/templates.
- Слишком частый `flush()` в цикле.
- Изменения через raw SQL не синхронизированы с identity map.

Пример batch:

```php
foreach ($rows as $i => $row) {
    $entityManager->persist(UserImport::fromRow($row));

    if (($i % 500) === 0) {
        $entityManager->flush();
        $entityManager->clear();
    }
}

$entityManager->flush();
$entityManager->clear();
```

Senior-пояснение: Doctrine дает сильную модель Unit of Work и identity map, но требует дисциплины
с transaction boundaries, batch processing и loading strategies.

## Unit of Work и Identity Map

Unit of Work отвечает за отслеживание объектов и запись изменений пачкой.

Что сказать на интервью:

- Unit of Work уменьшает количество ручных SQL operations и позволяет ORM вычислять порядок INSERT/UPDATE/DELETE.
- Identity Map предотвращает ситуацию, когда одна строка БД представлена несколькими объектами с разным state.
- Недостаток: память растет вместе с количеством managed entities; long-running jobs требуют очистки context.

Практический пример проблемы:

- В Doctrine загрузили `User#1`.
- Потом raw SQL изменил `users.name`.
- EntityManager все еще вернет старый объект из identity map.
- Нужно `refresh()`, `clear()` или не смешивать уровни доступа без контроля.

## Lazy/Eager Loading и N+1

Как объяснять:

- Lazy loading откладывает запрос до обращения к relation.
- Eager loading заранее загружает relation одним или несколькими запросами.
- N+1 появляется, когда на N объектов выполняется N дополнительных запросов для связей.

Как искать:

- Symfony Profiler;
- Doctrine SQL logger/profiler;
- тесты с ограничением количества queries для критичных сценариев.

Как чинить:

- Fetch joins;
- explicit joins;
- DTO projections;
- `EXTRA_LAZY` для коллекций;
- read models;
- не сериализовать entity напрямую без явного shape.

## Transactions и Consistency

```php
$entityManager->wrapInTransaction(function () use ($command): void {
    $order = Order::fromCommand($command);
    $this->entityManager->persist($order);
});
```

Важно:

- `flush()` внутри transaction синхронизирует Unit of Work с connection.
- Domain events лучше публиковать после commit или через outbox.
- Optimistic locking доступен через version fields.

Питфоллы:

- Считать transaction заменой идемпотентности.
- Публиковать message до commit.
- Делать retries без понимания, какие операции уже выполнены.

## Migrations

- Обычно Doctrine migrations генерируются diff-ом metadata/schema, но diff нужно ревьюить.
- Entity mapping не равно безопасная миграция.
- Для zero-downtime нужны expand/contract подходы.

Checklist безопасной миграции:

- Добавление nullable column или column с default без full table rewrite, если СУБД это поддерживает.
- Backfill отдельным job/batch.
- Код умеет работать со старой и новой схемой на время deploy.
- Constraint/index добавляются с учетом lock-ов.
- Rollback plan описан явно.

## Messenger, Messages, Workers

Основные элементы:

- Message - DTO команды/события.
- Handler обрабатывает message.
- Transports: Doctrine, AMQP, Redis, SQS через integrations.
- Middleware pipeline может добавлять transaction, validation, routing и retries.
- Failure transport играет роль DLQ.

Питфоллы:

- Смешивать message DTO и Doctrine entity.
- Не настраивать retry strategy/failure transport.
- Не очищать EntityManager между сообщениями при custom workers.
- Делать handler неидемпотентным.

## Retries, Idempotency и Failure Transport

Senior-позиция:

- Любая distributed операция может выполниться 0, 1 или несколько раз.
- Retry безопасен только при идемпотентной операции или при наличии idempotency key.
- Failure transport нужен не для игнорирования ошибок, а для диагностики и controlled replay.

Идемпотентность на практике:

- Unique key на business operation: `payment_id`, `external_event_id`, `order_id + step`.
- Таблица processed events/inbox.
- Outbox pattern для надежной публикации сообщений после DB commit.
- Проверка текущего state перед выполнением side effect.
- External API idempotency key, если поддерживается.

Checklist для handler/message:

- Можно ли безопасно обработать message дважды?
- Что будет, если worker умрет после DB commit, но до ack?
- Что будет, если external API ответил timeout, но операцию выполнил?
- Есть ли уникальный ключ операции?
- Где видны failed messages и кто их replay-ит?
- Есть ли backoff с jitter для внешних сервисов?

## Testing Specifics

Подробная стратегия тестирования вынесена в `03-php-testing-frameworks-phpunit-codeception-pest.md`.
Для Symfony важно помнить фреймворковые инструменты.

Что использовать:

- `KernelTestCase` для service/container integration.
- `WebTestCase` для HTTP.
- Test container для замены services.
- Test environment config.
- Messenger test transport или in-memory transport.
- Doctrine fixtures/foundry.
- DAMA Doctrine Test Bundle, transactional тесты или isolated test DB.
- HttpClient mock responses.
- Service decoration/test doubles in test container.

Практические советы:

- Доменные сервисы тестировать без kernel, если возможно.
- Container boot дорогой, не использовать его для чистых unit-тестов.
- Repository тестировать с реальной БД.
- WebTestCase хорош для API и security flows.
- Messenger handler тестировать напрямую и через transport отдельно.

Питфоллы:

- Boot kernel в каждом unit test без необходимости.
- Использовать real external services вместо test doubles.
- Не проверять compiler/container конфигурацию в integration tests.
- Слишком много `KernelTestCase` делает suite медленным и маскирует плохую декомпозицию.

## Вопросы и Короткие Ответы

**Как Symfony выбирает controller и аргументы?**  
Routing listener определяет controller, затем argument/value resolvers собирают аргументы из request attributes,
services, entity mapping, request body и других источников.

**Чем autowiring отличается от autoconfiguration?**  
Autowiring подставляет зависимости по типам.
Autoconfiguration автоматически добавляет tags/configuration по interfaces/attributes/base classes.

**Почему private services by default?**  
Чтобы container мог оптимизировать граф, удалить неиспользуемые services и заставить код использовать
explicit DI вместо service locator.

**Что такое Unit of Work?**  
Компонент ORM, который отслеживает managed entities, вычисляет changesets и синхронизирует изменения с БД при `flush()`.

**Что такое Identity Map?**  
Кэш соответствия `class + id -> object instance` внутри EntityManager, чтобы одна строка БД была представлена одним объектом.

**Где должна жить бизнес-логика?**  
Не в middleware/controller/entity lifecycle hooks по умолчанию.
Обычно в application services/use cases/domain services/entities/value objects, в зависимости от архитектуры.

**Как проектировать retries?**  
Считать, что операция может повториться. Нужны idempotency key, unique constraints, state checks,
backoff, failure transport и наблюдаемость.

**Что опасно в ORM events/listeners?**  
Скрытые side effects, порядок вызова, выполнение внутри transaction, рекурсивный `flush`,
сложность тестирования и профилирования.

**Как объяснить lazy loading senior-аудитории?**  
Это удобный IO-on-property-access. Он снижает boilerplate, но прячет запросы и может разрушить latency
при сериализации, шаблонах и циклах.

## Мини-Практика

### 1. Нарисовать lifecycle

`index.php -> Kernel -> HttpKernel -> events -> router -> controller resolver -> argument resolver -> response events`.

Проверь себя: где подключаются DI, routing, auth, exception handling и termination?

### 2. Найти N+1

Сценарий: serializer или template обращается к lazy relation внутри цикла.

Решения: fetch join, explicit join, DTO projection, read model, query count assertion в тесте.

### 3. Спроектировать idempotent handler

Сценарий: отправить welcome coupon пользователю после регистрации.

Решение:

- Message содержит `userId`, не Doctrine entity.
- Таблица `issued_coupons` имеет unique index на `user_id + coupon_type`.
- Перед external call проверяется, не был ли coupon уже выдан.
- External API получает idempotency key `welcome-coupon:{userId}`.
- Retry strategy использует exponential backoff.
- После исчерпания retries message попадает в failure transport.

### 4. Проверить transaction boundary

- Где начинается и заканчивается transaction?
- Есть ли external call внутри transaction?
- Что произойдет при exception после частичного side effect?
- Можно ли повторить операцию?
- Нужен ли outbox?

## Финальный Чеклист

- Умею объяснить lifecycle Symfony без деталей исходников.
- Понимаю EventDispatcher и HttpKernel events.
- Могу объяснить Symfony compiled container, private services, autowiring и autoconfiguration.
- Знаю, где bundles/compiler passes расширяют приложение.
- Умею найти и исправить N+1 в Doctrine.
- Могу объяснить Unit of Work, Identity Map и lazy proxies.
- Понимаю transaction boundaries и outbox.
- Могу спроектировать idempotent Messenger handler с retries и failure transport.
- Знаю, чем WebTestCase/KernelTestCase отличаются от unit tests.

## Ссылки

Официальная документация:

- Symfony Documentation: <https://symfony.com/doc/current/index.html>
- Symfony HttpKernel Component: <https://symfony.com/doc/current/components/http_kernel.html>
- Symfony Event Dispatcher: <https://symfony.com/doc/current/event_dispatcher.html>
- Symfony Service Container: <https://symfony.com/doc/current/service_container.html>
- Symfony Bundles: <https://symfony.com/doc/current/bundles.html>
- Symfony Routing: <https://symfony.com/doc/current/routing.html>
- Symfony Validation: <https://symfony.com/doc/current/validation.html>
- Symfony Configuration: <https://symfony.com/doc/current/configuration.html>
- Symfony Secrets: <https://symfony.com/doc/current/configuration/secrets.html>
- Symfony Messenger: <https://symfony.com/doc/current/messenger.html>
- Symfony Messenger Testing: <https://symfony.com/doc/current/messenger.html#testing>
- Symfony Testing: <https://symfony.com/doc/current/testing.html>
- Doctrine ORM Documentation: <https://www.doctrine-project.org/projects/orm.html>
- Doctrine ORM Unit of Work internals: <https://www.doctrine-project.org/projects/doctrine-orm/en/current/reference/unitofwork.html>
- Doctrine ORM Working with Objects: <https://www.doctrine-project.org/projects/doctrine-orm/en/current/reference/working-with-objects.html>
- Doctrine Migrations: <https://www.doctrine-project.org/projects/migrations.html>

Статьи и паттерны:

- Martin Fowler, Data Mapper: <https://martinfowler.com/eaaCatalog/dataMapper.html>
- Martin Fowler, Unit of Work: <https://martinfowler.com/eaaCatalog/unitOfWork.html>
- Martin Fowler, Identity Map: <https://martinfowler.com/eaaCatalog/identityMap.html>
- Microsoft Azure Architecture Center, Retry pattern: <https://learn.microsoft.com/en-us/azure/architecture/patterns/retry>
- Microsoft Azure Architecture Center, Competing Consumers pattern: <https://learn.microsoft.com/en-us/azure/architecture/patterns/competing-consumers>
- Microservices.io, Transactional Outbox: <https://microservices.io/patterns/data/transactional-outbox.html>
- Stripe API idempotent requests: <https://docs.stripe.com/api/idempotent_requests>
