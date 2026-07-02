# Laravel: internals, DI, middleware, Eloquent, queues

Цель: быстро повторить Laravel перед интервью уровня Middle+/Senior/Lead PHP developer.

Фокус: как Laravel исполняет HTTP-запрос, строит зависимости, работает с Eloquent, транзакциями,
очередями и тестами.

## Быстрый Senior-Ответ

Laravel строится вокруг service container, service providers и HTTP kernel/middleware pipeline.

Большая часть магии - это container bindings, facades как static proxy к container service,
route model binding, Eloquent Active Record, queue workers и удобные testing fakes.

На senior-уровне важно говорить не только про API фреймворка, но и про границы транзакций, N+1,
долгоживущие процессы, идемпотентность jobs и тестируемость.

## Request Lifecycle

Упрощенный путь HTTP-запроса:

1. `public/index.php` загружает Composer autoload и создает приложение из `bootstrap/app.php`.
2. Приложение поднимает HTTP Kernel.
3. Kernel bootstrap-ит окружение, конфигурацию, service providers, exception handling и facades.
4. Запрос проходит через global middleware.
5. Router находит route, применяет route middleware и вызывает controller/closure.
6. Controller dependencies resolve-ятся через service container.
7. Response проходит обратно через middleware.
8. Kernel завершает request и вызывает terminable middleware.

Практические детали:

- `bootstrap/app.php` в новых версиях Laravel стал центральным местом конфигурации routing, middleware и exceptions.
- `config:cache` объединяет конфиги в один PHP-файл; после этого runtime-код не должен читать `.env` напрямую.
- Facade не является static-сервисом в прямом смысле: это статический proxy к объекту из container.
- В Octane, RoadRunner и Swoole нельзя мыслить request-scoped как в PHP-FPM: singleton-состояние живет между запросами.

Короткий ответ: front controller грузит autoload, создает app, HTTP Kernel bootstrap-ит config/providers.
Request идет через middleware pipeline, router находит route, container resolve-ит controller,
response проходит обратно.

## Service Container и DI

Что надо знать:

- Container умеет auto-resolve классы через reflection, если зависимости типизированы concrete-классами.
- Interfaces и абстракции нужно явно bind-ить через `bind`, `singleton`, `scoped` или contextual binding.
- `singleton` живет весь lifecycle приложения; в FPM это обычно один request, в long-running worker намного дольше.
- `scoped` полезен для request/job lifecycle в long-running процессах.
- Service providers регистрируют bindings в `register()` и выполняют boot-логику в `boot()`.

Типичные ошибки:

- Держать mutable request/user state в singleton.
- Инжектить весь container вместо конкретных зависимостей.
- Использовать facade там, где нужен явный контракт и тестируемость.
- Делать тяжелую работу в `register()` вместо lazy binding.

Пример:

```php
// AppServiceProvider::register()
$this->app->bind(PaymentGateway::class, StripePaymentGateway::class);

$this->app->when(AdminReportService::class)
    ->needs(CacheRepository::class)
    ->give(fn ($app) => $app->make('cache')->store('redis'));
```

Senior-пояснение: binding интерфейса отделяет use case от инфраструктуры.
Contextual binding позволяет не плодить фабрики ради одного отличающегося backend-а.

## Service Providers

Provider отвечает за регистрацию и bootstrapping части приложения.

- `register()`: только регистрация bindings/config merge.
- `boot()`: routes, views, event listeners, publishing, macros, policies.
- Deferred providers загружаются при первом запросе нужного service.

Питфоллы:

- Побочные эффекты в `register()`.
- Зависимость boot-порядка между provider-ами.
- Слишком много логики в `AppServiceProvider`.

## Middleware Pipeline

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

Senior-пояснение: middleware подходит для HTTP concerns, но не должен превращаться в слой доменной логики.

## Routing и Controllers

- Routes описываются в route files или через attributes/packages.
- Route model binding может автоматически подгружать model по `{id}` или slug.
- Middleware можно назначать global, group или route-level.
- Controller dependencies resolve-ятся container-ом.

Питфоллы:

- Неявный route model binding может скрыть дополнительные SQL-запросы.
- `apiResource` удобен, но иногда провоцирует CRUD-мышление вместо use-case endpoints.
- Большие controllers становятся transaction script без явной архитектуры.

## Validation

- `FormRequest` хорошо отделяет HTTP validation/authorization от controller.
- Rules могут быть строковыми, объектными и closure-based.
- Для сложной доменной инвариантности validation layer недостаточен: нужна проверка в domain/application service.

Питфоллы:

- Полагаться на validation как на единственную защиту от race conditions.
- Делать database-heavy custom rules без индексов и batching.
- Валидировать nested arrays без лимитов размера.

## Configuration и Env

- `.env` читается на bootstrap stage.
- После `config:cache` приложение должно читать значения через `config()`, не через `env()`.
- Config files должны быть deterministic и serializable.

Питфоллы:

- Использовать `env()` в runtime-коде.
- Кэшировать config и забывать очистить после deploy.
- Хранить секреты в репозитории.

Короткий ответ: после `config:cache` `.env` не должен быть runtime-источником значений; приложение должно читать `config()`.

## Eloquent Internals

Eloquent - Active Record ORM: model совмещает state, persistence API, relations и query builder.

Что важно:

- Model instance представляет строку таблицы.
- Relations lazy-load-ятся при обращении к property.
- `with()` делает eager loading.
- `load()` догружает relations для уже полученных моделей.
- Accessors, mutators и casts могут скрывать CPU/IO стоимость.
- Global scopes влияют на все запросы модели.
- Events и observers могут выполнять побочные эффекты при `save()`/`delete()`.

Питфоллы:

- N+1 из-за lazy loading в цикле.
- Mass assignment без понимания `$fillable`/`$guarded`.
- Непредсказуемые global scopes.
- События модели, отправляющие внешние запросы внутри transaction.
- `save()` в цикле вместо bulk operations.
- Утечки памяти при обработке больших наборов без `chunkById`, `cursor`, `lazyById`.

Пример N+1:

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

Senior-пояснение: Eloquent удобен для CRUD и application-level data access, но его Active Record природа
может смешивать доменную модель и persistence. На больших доменах стоит явно отделять use cases,
DTO/read models и transaction boundaries.

## Lazy/Eager Loading и N+1

Как объяснять:

- Lazy loading откладывает запрос до обращения к relation.
- Eager loading заранее загружает relation одним или несколькими запросами.
- N+1 появляется, когда на N объектов выполняется N дополнительных запросов для связей.

Как искать:

- Laravel Debugbar;
- Telescope;
- query log;
- database profiler;
- тесты с ограничением количества queries для критичных сценариев.

Как чинить:

- `with`;
- `withCount`;
- `loadMissing`;
- запрет lazy loading в non-production;
- явный response shape для API.

## Transactions и Consistency

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
- Для race conditions использовать unique constraints, row locks и optimistic locking patterns.

Питфоллы:

- Считать transaction заменой идемпотентности.
- Dispatch job до commit.
- Делать retries без понимания, какие операции уже выполнены.

Короткий ответ: `afterCommit()` нужен, когда worker должен увидеть уже committed state,
а не промежуточные данные текущей транзакции.

## Migrations

- PHP-классы описывают schema changes.
- Есть `up/down`, schema builder и raw SQL при необходимости.
- В production важно избегать долгих blocking migrations.

Checklist безопасной миграции:

- Добавление nullable column или column с default без full table rewrite, если СУБД это поддерживает.
- Backfill отдельным job/batch.
- Код умеет работать со старой и новой схемой на время deploy.
- Constraint/index добавляются с учетом lock-ов.
- Rollback plan описан явно.

## Queues, Jobs, Workers

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

## Retries, Idempotency и DLQ

Senior-позиция:

- Любая distributed операция может выполниться 0, 1 или несколько раз.
- Retry безопасен только при идемпотентной операции или при наличии idempotency key.
- DLQ/failure storage нужен не для игнорирования ошибок, а для диагностики и controlled replay.

Идемпотентность на практике:

- Unique key на business operation: `payment_id`, `external_event_id`, `order_id + step`.
- Таблица processed events/inbox.
- Outbox pattern для надежной публикации сообщений после DB commit.
- Проверка текущего state перед выполнением side effect.
- External API idempotency key, если поддерживается.

Checklist для job:

- Можно ли безопасно выполнить job дважды?
- Что будет, если worker умрет после DB commit, но до ack?
- Что будет, если external API ответил timeout, но операцию выполнил?
- Есть ли уникальный ключ операции?
- Где видны failed jobs и кто их replay-ит?
- Есть ли backoff с jitter для внешних сервисов?

## Testing Specifics

Подробная стратегия тестирования вынесена в `03-php-testing-frameworks-phpunit-codeception-pest.md`.
Для Laravel важно помнить фреймворковые инструменты.

Что использовать:

- Feature tests для HTTP endpoints.
- Unit tests для чистой доменной/application логики.
- `RefreshDatabase` и `DatabaseTransactions` для изоляции БД.
- Model factories и states.
- `actingAs`, `postJson`, `assertJsonPath`.
- `Queue::fake()`, `Bus::fake()`, `Event::fake()`, `Mail::fake()`, `Notification::fake()`, `Storage::fake()`.
- HTTP client fakes.
- `withoutExceptionHandling()` точечно для диагностики.
- Parallel testing при корректной изоляции БД/кэша.

Практические советы:

- Feature tests часто ценнее изолированных controller unit tests.
- Не мокать Eloquent без необходимости.
- Доменную логику выносить из controller/job в сервисы и тестировать отдельно.
- Проверять policies/authorization отдельно или через API сценарии.
- Для jobs тестировать dispatch и handler отдельно.

Питфоллы:

- Overmocking facades вместо проверки observable behavior.
- Тесты проходят только из-за SQLite, но падают на MySQL/PostgreSQL.
- Проверять implementation details вместо contract/outcome.
- `Event::fake()` может отключить observers/listeners, которые нужны для поведения.

## Вопросы и Короткие Ответы

**Что происходит от `index.php` до controller?**  
Front controller грузит autoload, создает app, HTTP Kernel bootstrap-ит config/providers.
Request идет через middleware pipeline, router находит route, container resolve-ит controller,
response проходит обратно.

**Facade - это плохо?**  
Не само по себе. Facade - удобный static proxy к container service.
Риск в скрытых зависимостях и усложнении тестирования. В domain/application коде лучше явный DI.

**Почему нельзя `env()` в runtime?**  
После `config:cache` `.env` не должен читаться как источник runtime-значений. Runtime-код должен читать `config()`.

**Как избежать N+1 в Eloquent?**  
Использовать `with`, `load`, `loadMissing`, `withCount`, профилировать SQL.
Запретить lazy loading в dev/test, проектировать response shape явно.

**Где должна жить бизнес-логика?**  
Не в middleware/controller/entity lifecycle hooks по умолчанию.
Обычно в application services/use cases/domain services/entities/value objects, в зависимости от архитектуры.

**Как проектировать retries?**  
Считать, что операция может повториться. Нужны idempotency key, unique constraints, state checks, backoff, DLQ и наблюдаемость.

**Что опасно в ORM events/listeners?**  
Скрытые side effects, порядок вызова, выполнение внутри transaction, рекурсивные `flush`/`save`,
сложность тестирования и профилирования.

**Как объяснить lazy loading senior-аудитории?**  
Это удобный IO-on-property-access. Он снижает boilerplate, но прячет запросы и может разрушить latency
при сериализации, шаблонах и циклах.

## Мини-Практика

### 1. Нарисовать lifecycle

`index.php -> app -> kernel bootstrap -> middleware -> router -> controller -> response -> terminate`.

Проверь себя: где подключаются DI, routing, auth, exception handling и termination?

### 2. Найти N+1

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
- После исчерпания retries job попадает в DLQ/failure storage.

### 4. Проверить transaction boundary

- Где начинается и заканчивается transaction?
- Есть ли external call внутри transaction?
- Что произойдет при exception после частичного side effect?
- Можно ли повторить операцию?
- Нужен ли outbox?

## Финальный Чеклист

- Умею объяснить lifecycle Laravel без деталей исходников.
- Понимаю middleware pipeline.
- Могу объяснить container bindings, contextual binding, `singleton` и `scoped`.
- Знаю, где service providers расширяют приложение.
- Умею найти и исправить N+1.
- Понимаю transaction boundaries, `afterCommit` и outbox.
- Могу спроектировать idempotent queue job с retries и DLQ.
- Знаю, чем feature/integration tests отличаются от unit tests в Laravel.

## Ссылки

Официальная документация:

- Laravel Documentation: <https://laravel.com/docs>
- Laravel Lifecycle: <https://laravel.com/docs/lifecycle>
- Laravel Service Container: <https://laravel.com/docs/container>
- Laravel Service Providers: <https://laravel.com/docs/providers>
- Laravel Middleware: <https://laravel.com/docs/middleware>
- Laravel Routing: <https://laravel.com/docs/routing>
- Laravel Validation: <https://laravel.com/docs/validation>
- Laravel Configuration: <https://laravel.com/docs/configuration>
- Laravel Eloquent ORM: <https://laravel.com/docs/eloquent>
- Laravel Eloquent Relationships: <https://laravel.com/docs/eloquent-relationships>
- Laravel Database Transactions: <https://laravel.com/docs/database#database-transactions>
- Laravel Migrations: <https://laravel.com/docs/migrations>
- Laravel Queues: <https://laravel.com/docs/queues>
- Laravel Horizon: <https://laravel.com/docs/horizon>
- Laravel Testing: <https://laravel.com/docs/testing>
- Laravel HTTP Tests: <https://laravel.com/docs/http-tests>
- Laravel Mocking/Fakes: <https://laravel.com/docs/mocking>

Статьи и паттерны:

- Martin Fowler, Active Record: <https://martinfowler.com/eaaCatalog/activeRecord.html>
- Martin Fowler, Unit of Work: <https://martinfowler.com/eaaCatalog/unitOfWork.html>
- Martin Fowler, Identity Map: <https://martinfowler.com/eaaCatalog/identityMap.html>
- Microsoft Azure Architecture Center, Retry pattern: <https://learn.microsoft.com/en-us/azure/architecture/patterns/retry>
- Microsoft Azure Architecture Center, Competing Consumers pattern: <https://learn.microsoft.com/en-us/azure/architecture/patterns/competing-consumers>
- Microservices.io, Transactional Outbox: <https://microservices.io/patterns/data/transactional-outbox.html>
- Stripe API idempotent requests: <https://docs.stripe.com/api/idempotent_requests>
