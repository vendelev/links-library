# За 1 день: Laravel/Symfony internals, DI, middleware, ORM, queues

Цель: быстро освежить прагматичные детали Laravel и Symfony перед интервью уровня Lead/Senior PHP developer. Фокус на том, как фреймворки реально исполняют запрос, строят зависимости, работают с ORM, очередями и тестами, а также на типичных ловушках, которые ожидают услышать на собеседовании.

## Как пользоваться за 1 день

1. Прочитать блоки `Request lifecycle`, `DI`, `Middleware/Event Dispatcher/Kernel` и `ORM`.
2. Проговорить вслух короткие ответы из раздела `Вопросы и короткие senior-ответы`.
3. Решить мини-практику: нарисовать lifecycle запроса, найти N+1, спроектировать idempotent job.
4. Перед интервью пройти чеклист в конце.

## Request lifecycle

### Laravel

Упрощенный путь HTTP-запроса:

1. `public/index.php` загружает Composer autoload и создает приложение из `bootstrap/app.php`.
2. Приложение поднимает HTTP Kernel.
3. Kernel bootstrap-ит окружение, конфигурацию, service providers, exception handling, facades.
4. Запрос проходит через global middleware.
5. Router находит route, применяет route middleware и вызывает controller/closure.
6. Controller использует container для dependency injection.
7. Response проходит обратно через middleware.
8. Kernel завершает request и вызывает terminable middleware.

Ключевая мысль для интервью: Laravel строит приложение вокруг service container и service providers. Большая часть магии: deferred/lazy services, facades как proxy к container bindings, route model binding, middleware pipeline.

Практические детали:

- `bootstrap/app.php` в новых версиях Laravel стал центральным местом конфигурации routing, middleware, exceptions.
- `config:cache` объединяет конфиги в один PHP-файл; после этого нельзя ожидать runtime-чтение `.env` в приложении.
- Facade не является static-сервисом в прямом смысле: это статический прокси к объекту из container.
- В Octane/RoadRunner/Swoole нельзя мыслить request-scoped как в FPM: singleton-состояние живет между запросами.

### Symfony

Упрощенный путь HTTP-запроса:

1. Front controller `public/index.php` создает Kernel.
2. Kernel загружает bundles, config, container.
3. HttpKernel обрабатывает `Request` через Event Dispatcher.
4. `kernel.request`: routing, locale, security listeners и другие request listeners.
5. Router определяет controller и route attributes.
6. Argument resolver собирает аргументы controller action.
7. Controller возвращает `Response` или данные, преобразуемые listeners.
8. `kernel.response` модифицирует response.
9. `kernel.terminate` выполняет post-response задачи.

Ключевая мысль для интервью: Symfony явно построен вокруг HttpKernel, EventDispatcher и скомпилированного DI container. Многие компоненты независимы и используются Laravel тоже: Console, HttpFoundation, Routing, EventDispatcher, VarDumper.

Практические детали:

- Symfony container компилируется, оптимизируется и удаляет неиспользуемые private services.
- Controller invocation идет через resolver-ы, а не просто `new Controller()`.
- Большая часть расширяемости находится в events/listeners/subscribers/compiler passes.

## Service Container и DI

### Laravel container

Что надо знать:

- Container умеет auto-resolve классы через reflection, если зависимости типизированы concrete-классами.
- Interfaces/абстракции нужно явно bind-ить: `bind`, `singleton`, `scoped`, contextual binding.
- `singleton` живет весь lifecycle приложения; в FPM это обычно один request, в long-running worker намного дольше.
- `scoped` полезен для request/job lifecycle в long-running процессах.
- Service providers регистрируют bindings в `register()` и выполняют boot-логику в `boot()`.

Типичные ошибки:

- Держать mutable request/user state в singleton.
- Инжектить весь container вместо конкретных зависимостей.
- Использовать facade там, где нужен явный контракт и тестируемость.
- Делать тяжелую работу в `register()` вместо lazy binding.

Мини-пример:

```php
// AppServiceProvider::register()
$this->app->bind(PaymentGateway::class, StripePaymentGateway::class);

$this->app->when(AdminReportService::class)
    ->needs(CacheRepository::class)
    ->give(fn ($app) => $app->make('cache')->store('redis'));
```

Senior-пояснение: binding интерфейса отделяет use case от инфраструктуры, а contextual binding позволяет не плодить фабрики ради одного отличающегося backend-а.

### Symfony DI

Что надо знать:

- Services обычно private by default.
- Autowiring подбирает зависимости по type-hint.
- Autoconfiguration добавляет tags/interfaces автоматически: commands, event subscribers, validators и т.д.
- Attributes помогают конфигурировать injection, routes, listeners, autowire aliases.
- Compiler pass позволяет менять container на этапе компиляции.

Типичные ошибки:

- Делать все services public, чтобы доставать их из container.
- Прятать зависимости через service locator без необходимости.
- Игнорировать compile-time ошибки container и пытаться решать их runtime-хаками.
- Не понимать разницу между autowiring alias и concrete service id.

Мини-пример:

```yaml
# config/services.yaml
services:
  App\:
    resource: '../src/'
    exclude: '../src/{DependencyInjection,Entity,Kernel.php}'

  App\Billing\PaymentGatewayInterface:
    alias: App\Billing\StripePaymentGateway
```

Senior-пояснение: Symfony DI стремится к compile-time проверке графа зависимостей. Это снижает runtime-сюрпризы и делает приложение более предсказуемым.

## Service Providers и Bundles

### Laravel Service Providers

Provider отвечает за регистрацию и bootstrapping части приложения.

- `register()`: только регистрация bindings/config merge.
- `boot()`: routes, views, event listeners, publishing, macros, policies.
- Deferred providers загружаются при первом запросе нужного service.

Питфоллы:

- Побочные эффекты в `register()`.
- Зависимость boot-порядка между provider-ами.
- Слишком много логики в `AppServiceProvider`.

### Symfony Bundles

Bundle - модуль расширения приложения или reusable package.

- `Bundle` class подключает расширение.
- `Extension` загружает config и services.
- Compiler passes модифицируют container до компиляции.
- Recipes/Flex помогают устанавливать config для bundles.

Питфоллы:

- Тащить bundle ради маленькой функции.
- Писать business logic внутри bundle extension.
- Не документировать extension config tree.

## Middleware, Event Dispatcher и Kernel

### Laravel middleware pipeline

Middleware оборачивает request/response:

```php
public function handle(Request $request, Closure $next): Response
{
    if (! $request->user()) {
        abort(401);
    }

    return $next($request);
}
```

Где применять:

- auth/authz на уровне HTTP;
- rate limiting;
- locale/tenant resolution;
- correlation id/request id;
- security headers;
- request normalization.

Не стоит делать:

- business transactions;
- сложную доменную валидацию;
- отправку внешних запросов без крайней необходимости;
- тяжелые операции, блокирующие каждый request.

### Symfony HttpKernel events

Основные события:

- `kernel.request`: до controller, можно изменить request или вернуть response.
- `kernel.controller`: после выбора controller.
- `kernel.controller_arguments`: после resolution аргументов.
- `kernel.view`: если controller вернул не Response.
- `kernel.response`: перед отправкой response.
- `kernel.exception`: обработка исключений.
- `kernel.terminate`: после отправки response.

Senior-пояснение: Laravel чаще мыслится pipeline middleware, Symfony - event-driven kernel. В обоих случаях важно не смешивать HTTP concerns с доменной логикой.

## Routing и Controllers

### Laravel

- Routes описываются в route files или через attributes/packages.
- Route model binding может автоматически подгружать model по `{id}`/slug.
- Middleware можно назначать global/group/route-level.
- Controller dependencies resolve-ятся container-ом.

Питфоллы:

- Неявный route model binding может скрыть дополнительные запросы.
- `apiResource` удобен, но иногда провоцирует CRUD-мышление вместо use-case endpoints.
- Большие controllers становятся transaction script без явной архитектуры.

### Symfony

- Routing обычно через attributes, YAML/XML/PHP config.
- Controller arguments resolve-ятся через value resolvers.
- Param converters в новых проектах часто заменяются MapEntity/value resolvers.
- Controller должен быть тонким: orchestration, validation, mapping response.

Питфоллы:

- Смешивать entity, form, command и response DTO в одном объекте.
- Перегружать controller security expressions бизнес-правилами.
- Возвращать Doctrine entities напрямую в публичный API без контроля сериализации.

## Validation

### Laravel

- `FormRequest` хорошо отделяет HTTP validation/authorization от controller.
- Rules могут быть строковыми, объектными, closure-based.
- Для сложной доменной инвариантности validation layer недостаточен: нужна проверка в domain/application service.

Питфоллы:

- Полагаться на validation как на единственную защиту от race conditions.
- Делать database-heavy custom rules без индексов и batching.
- Валидировать nested arrays без лимитов размера.

### Symfony

- Validator component использует constraints через attributes/YAML/XML/PHP.
- Validation groups помогают разным сценариям, но могут усложнить модель.
- Forms полезны для HTML, но для API часто проще DTO + Validator.

Питфоллы:

- Считать constraint доменным правилом, когда правило должно жить в aggregate/service.
- Слишком сложные validation groups вместо явных input DTO.

## Configuration и env

### Laravel

- `.env` читается на bootstrap stage.
- После `config:cache` приложение должно читать значения через `config()`, не через `env()`.
- Config files должны быть deterministic и serializable.

Питфоллы:

- Использовать `env()` в runtime-коде.
- Кэшировать config и забывать очистить после deploy.
- Хранить секреты в репозитории.

### Symfony

- Env vars могут использоваться в config через `%env(...)%`.
- Secrets component подходит для encrypted secrets.
- Config normalizer/processor в bundle extension валидирует структуру config.
- Cache warmup важен для production deploy.

Питфоллы:

- Путать build-time и runtime env.
- Делать dynamic config, который ломает container compilation.
- Не различать параметры container и env processors.

## ORM internals и pitfalls

## Eloquent

Eloquent - Active Record ORM: model совмещает state, persistence API, relations и query builder.

Что важно:

- Model instance представляет строку таблицы.
- Relations lazy-load-ятся при обращении к property.
- `with()` делает eager loading.
- `load()` догружает relations для уже полученных моделей.
- Accessors/mutators/casts могут скрывать CPU/IO стоимость.
- Global scopes влияют на все запросы модели.
- Events/observers могут выполнять побочные эффекты при save/delete.

Питфоллы:

- N+1 из-за lazy loading в цикле.
- Mass assignment без `$fillable`/`$guarded` понимания.
- Непредсказуемые global scopes.
- События модели, отправляющие внешние запросы внутри transaction.
- `save()` в цикле вместо bulk operations.
- Утечки памяти при обработке больших наборов без `chunkById`, `cursor`, `lazyById`.

Мини-пример N+1:

```php
// Плохо: запросы authors выполняются в цикле.
$books = Book::query()->latest()->take(50)->get();

foreach ($books as $book) {
    echo $book->author->name;
}

// Лучше.
$books = Book::query()
    ->with('author')
    ->latest()
    ->take(50)
    ->get();
```

Senior-пояснение: Eloquent удобен для CRUD и application-level data access, но его Active Record природа может смешивать доменную модель и persistence. На больших доменах стоит явно отделять use cases, DTO/read models и transaction boundaries.

## Doctrine

Doctrine ORM - Data Mapper ORM: entity обычно не знает о persistence API, EntityManager управляет состоянием.

Ключевые понятия:

- EntityManager - фасад к Unit of Work, repositories, connection.
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

Мини-пример batch:

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

Senior-пояснение: Doctrine дает сильную модель Unit of Work и identity map, но требует дисциплины с transaction boundaries, batch processing и loading strategies.

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

- Laravel Debugbar/Telescope, query log, database profiler.
- Symfony Profiler, Doctrine SQL logger/profiler.
- Тесты с ограничением количества queries для критичных сценариев.

Как чинить:

- Laravel: `with`, `withCount`, `loadMissing`, запрет lazy loading в non-production.
- Doctrine: fetch joins, explicit joins, DTO projections, `EXTRA_LAZY` для коллекций, read models.
- Для API: не сериализовать entity/model без явного shape.

## Transactions и consistency

### Laravel

```php
DB::transaction(function () use ($command) {
    $order = Order::createFromCommand($command);
    $order->save();

    ProcessOrderPayment::dispatch($order->id)->afterCommit();
});
```

Важно:

- Использовать `afterCommit()` для jobs/events, зависящих от committed state.
- Не держать transaction во время HTTP-вызовов к внешним сервисам.
- Для race conditions использовать unique constraints, row locks, optimistic locking patterns.

### Symfony/Doctrine

```php
$entityManager->wrapInTransaction(function () use ($command): void {
    $order = Order::fromCommand($command);
    $this->entityManager->persist($order);
});
```

Важно:

- `flush()` внутри transaction синхронизирует UoW с connection.
- Domain events лучше публиковать после commit или через outbox.
- Optimistic locking доступен через version fields.

Питфоллы:

- Считать transaction заменой идемпотентности.
- Dispatch job до commit.
- Делать retries без понимания, какие операции уже выполнены.

## Migrations

Laravel migrations:

- PHP-классы описывают schema changes.
- Есть `up/down`, schema builder, raw SQL при необходимости.
- В production важно избегать долгих blocking migrations.

Doctrine migrations:

- Обычно генерируются diff-ом metadata/schema, но diff нужно ревьюить.
- Entity mapping не равно безопасная миграция.
- Для zero-downtime нужны expand/contract подходы.

Checklist безопасной миграции:

- Добавление nullable column или column с default без full table rewrite, если СУБД это поддерживает.
- Backfill отдельным job/batch.
- Код умеет работать со старой и новой схемой на время deploy.
- Constraint/index добавляются с учетом lock-ов.
- Rollback plan описан явно.

## Queues, Jobs, Workers

### Laravel queues

Основные элементы:

- Job class содержит `handle()`.
- Drivers: database, Redis, SQS и другие.
- Worker выполняет jobs long-running процессом.
- Horizon управляет Redis queues и мониторингом.
- `tries`, `backoff`, `retryUntil`, `timeout`, `failed()` управляют retry/failure поведением.
- Failed jobs можно хранить и повторять.

Питфоллы:

- Передавать целые Eloquent models вместо id, не понимая сериализацию и stale data.
- Не задавать timeout/backoff.
- Не перезапускать workers после deploy: старый код остается в памяти.
- Хранить request-scoped state в singleton внутри worker.
- Не делать job идемпотентной.

### Symfony Messenger

Основные элементы:

- Message - DTO команды/события.
- Handler обрабатывает message.
- Transports: Doctrine, AMQP, Redis, SQS через integrations.
- Middleware pipeline может добавлять transaction, validation, routing, retries.
- Failure transport играет роль DLQ.

Питфоллы:

- Смешивать message DTO и Doctrine entity.
- Не настраивать retry strategy/failure transport.
- Не очищать EntityManager между сообщениями при custom workers.
- Делать handler неидемпотентным.

## Retries, idempotency и DLQ

Senior-позиция:

- Любая distributed операция может выполниться 0, 1 или несколько раз.
- Retry безопасен только при идемпотентной операции или при наличии idempotency key.
- DLQ/failure transport нужен не для игнорирования ошибок, а для диагностики и controlled replay.

Идемпотентность на практике:

- Unique key на business operation: `payment_id`, `external_event_id`, `order_id + step`.
- Таблица processed events/inbox.
- Outbox pattern для надежной публикации сообщений после DB commit.
- Проверка текущего state перед выполнением side effect.
- External API idempotency key, если поддерживается.

Мини-пример checklist для job:

- Можно ли безопасно выполнить job дважды?
- Что будет, если worker умрет после DB commit, но до ack?
- Что будет, если external API ответил timeout, но операцию выполнил?
- Есть ли уникальный ключ операции?
- Где видны failed jobs и кто их replay-ит?
- Есть ли backoff с jitter для внешних сервисов?

## Testing

### Laravel

Что использовать:

- Feature tests для HTTP endpoints.
- Unit tests для чистой доменной/application логики.
- `RefreshDatabase`/transactions для изоляции БД.
- `Queue::fake()`, `Bus::fake()`, `Event::fake()`, `Notification::fake()`.
- `withoutExceptionHandling()` точечно для диагностики.

Питфоллы:

- Overmocking facades вместо проверки observable behavior.
- Тесты проходят только из-за SQLite, но падают на MySQL/PostgreSQL.
- Проверять implementation details вместо contract/outcome.

### Symfony

Что использовать:

- `KernelTestCase` для service/container integration.
- `WebTestCase` для HTTP.
- Test container для замены services.
- Messenger test transport/in-memory transport.
- Doctrine fixtures/foundry, transactional тесты или isolated test DB.

Питфоллы:

- Boot kernel в каждом unit test без необходимости.
- Использовать real external services вместо test doubles.
- Не проверять compiler/container конфигурацию в integration tests.

## Вопросы и короткие senior-ответы

### Laravel

**Что происходит от `index.php` до controller?**  
Front controller грузит autoload, создает app, HTTP Kernel bootstrap-ит config/providers, request идет через middleware pipeline, router находит route, container resolve-ит controller, response проходит обратно.

**Facade - это плохо?**  
Не само по себе. Facade - удобный static proxy к container service. Риск в скрытых зависимостях и усложнении тестирования. В domain/application коде лучше явный DI.

**Почему нельзя `env()` в runtime?**  
После `config:cache` `.env` не должен читаться как источник runtime-значений. Runtime-код должен читать `config()`.

**Как избежать N+1 в Eloquent?**  
Использовать `with/load/loadMissing/withCount`, профилировать SQL, запретить lazy loading в dev/test, проектировать response shape явно.

**Когда использовать `afterCommit()`?**  
Когда job/event зависит от данных, которые должны быть committed. Иначе worker может увидеть несуществующее или старое состояние.

### Symfony

**Как Symfony выбирает controller и аргументы?**  
Routing listener определяет controller, затем argument/value resolvers собирают аргументы из request attributes, services, entity mapping, request body и других источников.

**Чем autowiring отличается от autoconfiguration?**  
Autowiring подставляет зависимости по типам. Autoconfiguration автоматически добавляет tags/configuration по interfaces/attributes/base classes.

**Почему private services by default?**  
Чтобы container мог оптимизировать граф, удалить неиспользуемые services и заставить код использовать explicit DI вместо service locator.

**Что такое Unit of Work?**  
Компонент ORM, который отслеживает managed entities, вычисляет changesets и синхронизирует изменения с БД при `flush()`.

**Что такое identity map?**  
Кэш соответствия `class + id -> object instance` внутри EntityManager, чтобы одна строка БД была представлена одним объектом.

### Общие вопросы

**Где должна жить бизнес-логика?**  
Не в middleware/controller/entity lifecycle hooks по умолчанию. Обычно в application services/use cases/domain services/entities/value objects, в зависимости от архитектуры.

**Как проектировать retries?**  
Считать, что операция может повториться. Нужны idempotency key, уникальные constraints, state checks, backoff, DLQ и наблюдаемость.

**Что опасно в ORM events/listeners?**  
Скрытые side effects, порядок вызова, выполнение внутри transaction, рекурсивные flush/save, сложность тестирования и профилирования.

**Как объяснить lazy loading senior-аудитории?**  
Это удобный IO-on-property-access. Он снижает boilerplate, но прячет запросы и может разрушить latency при сериализации, шаблонах и циклах.

## Мини-практика

### 1. Нарисовать lifecycle

Нарисуйте два потока:

- Laravel: `index.php -> app -> kernel bootstrap -> middleware -> router -> controller -> response -> terminate`.
- Symfony: `index.php -> Kernel -> HttpKernel -> events -> router -> controller resolver -> argument resolver -> response events`.

Проверьте себя: где подключаются DI, routing, auth, exception handling, termination?

### 2. Найти N+1

Код:

```php
$orders = Order::query()->latest()->limit(100)->get();

foreach ($orders as $order) {
    echo $order->customer->email;
    echo $order->items->count();
}
```

Ответ:

```php
$orders = Order::query()
    ->with('customer')
    ->withCount('items')
    ->latest()
    ->limit(100)
    ->get();
```

### 3. Спроектировать idempotent job

Сценарий: отправить welcome coupon пользователю после регистрации.

Решение:

- Job принимает `userId`, не весь model.
- Таблица `issued_coupons` имеет unique index на `user_id + coupon_type`.
- Перед external call проверяется, не был ли coupon уже выдан.
- External API получает idempotency key `welcome-coupon:{userId}`.
- Retry с exponential backoff.
- После исчерпания retries job попадает в DLQ/failure transport.

### 4. Проверить transaction boundary

Вопросы:

- Где начинается и заканчивается transaction?
- Есть ли external call внутри transaction?
- Что произойдет при exception после частичного side effect?
- Можно ли повторить операцию?
- Нужен ли outbox?

## Финальный чеклист перед интервью

- Умею объяснить lifecycle Laravel и Symfony без деталей исходников.
- Понимаю разницу middleware pipeline и HttpKernel events.
- Могу объяснить Laravel container bindings и Symfony compiled container.
- Знаю, где service providers/bundles расширяют приложение.
- Умею найти и исправить N+1.
- Могу объяснить Unit of Work, identity map, lazy proxies.
- Понимаю transaction boundaries и `afterCommit`/outbox.
- Могу спроектировать idempotent queue job с retries и DLQ.
- Знаю, чем feature/integration tests отличаются от unit tests в этих фреймворках.

## Ссылки

Официальная документация:

- Laravel Documentation: https://laravel.com/docs
- Laravel Service Container: https://laravel.com/docs/container
- Laravel Service Providers: https://laravel.com/docs/providers
- Laravel Middleware: https://laravel.com/docs/middleware
- Laravel Routing: https://laravel.com/docs/routing
- Laravel Validation: https://laravel.com/docs/validation
- Laravel Eloquent ORM: https://laravel.com/docs/eloquent
- Laravel Queues: https://laravel.com/docs/queues
- Laravel Testing: https://laravel.com/docs/testing
- Symfony Documentation: https://symfony.com/doc/current/index.html
- Symfony HttpKernel Component: https://symfony.com/doc/current/components/http_kernel.html
- Symfony Service Container: https://symfony.com/doc/current/service_container.html
- Symfony Event Dispatcher: https://symfony.com/doc/current/event_dispatcher.html
- Symfony Routing: https://symfony.com/doc/current/routing.html
- Symfony Validation: https://symfony.com/doc/current/validation.html
- Symfony Messenger: https://symfony.com/doc/current/messenger.html
- Symfony Testing: https://symfony.com/doc/current/testing.html
- Doctrine ORM Documentation: https://www.doctrine-project.org/projects/orm.html
- Doctrine ORM Unit of Work internals: https://www.doctrine-project.org/projects/doctrine-orm/en/current/reference/unitofwork.html
- Doctrine ORM Working with Objects: https://www.doctrine-project.org/projects/doctrine-orm/en/current/reference/working-with-objects.html
- Doctrine Migrations: https://www.doctrine-project.org/projects/migrations.html

Репутационные статьи и материалы:

- Martin Fowler: Unit of Work: https://martinfowler.com/eaaCatalog/unitOfWork.html
- Martin Fowler: Identity Map: https://martinfowler.com/eaaCatalog/identityMap.html
- Martin Fowler: Active Record: https://martinfowler.com/eaaCatalog/activeRecord.html
- Martin Fowler: Data Mapper: https://martinfowler.com/eaaCatalog/dataMapper.html
- Microsoft Azure Architecture Center: Retry pattern: https://learn.microsoft.com/en-us/azure/architecture/patterns/retry
- Microsoft Azure Architecture Center: Competing Consumers pattern: https://learn.microsoft.com/en-us/azure/architecture/patterns/competing-consumers
- Microservices.io: Transactional Outbox: https://microservices.io/patterns/data/transactional-outbox.html
- Stripe API idempotent requests: https://docs.stripe.com/api/idempotent_requests
