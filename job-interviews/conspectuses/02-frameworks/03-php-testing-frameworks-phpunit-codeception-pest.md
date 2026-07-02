# PHP Testing: PHPUnit, Codeception, Pest

Гайд для Lead/Senior PHP backend developer перед интервью.

Фокус: как объяснять тестирование практически, какие trade-off'ы видеть,
где чаще всего ломается стратегия тестов, и когда выбирать PHPUnit, Codeception или Pest.

## Быстрый Senior-Ответ

Тестирование в backend-проекте нужно не для процента покрытия, а для управляемого изменения системы.
Хорошая стратегия сочетает быстрые unit-тесты для бизнес-логики, интеграционные тесты для БД,
очередей и внешних адаптеров, функциональные HTTP/API тесты и ограниченное число e2e сценариев.

Senior ожидаемо говорит не только про инструменты, но и про стоимость поддержки: изоляция,
стабильные фикстуры, deterministic time/randomness, транзакции, CI, flaky-тесты,
mutation testing, контрактные тесты и тестируемость архитектуры.

## Пирамида Тестов

```text
        E2E
     Functional
   Integration / Contract
        Unit
```

Смысл пирамиды:

- Чем ниже уровень, тем тесты быстрее, дешевле и стабильнее.
- Чем выше уровень, тем больше уверенности в реальном поведении, но выше цена поддержки.
- Большинство бизнес-правил стоит проверять unit/integration тестами, а не UI/e2e.
- E2E должны покрывать только критические сценарии: регистрация, checkout, оплата, создание заказа, ключевой API flow.

Senior-комментарий: пирамида не догма. Для API/backend часто получается ромб или trophy:
много интеграционных/API тестов, если ценность системы в связке БД, ORM, HTTP и очередей.

## Уровни Тестирования

### Unit-Тесты

Проверяют малую единицу поведения без реальной БД, сети, файловой системы и очередей.

Хорошо подходят для:

- доменных сервисов;
- value objects;
- валидаторов;
- pricing/discount logic;
- state machines;
- permission policy logic.

Плюсы:

- быстрые;
- точная локализация ошибки;
- удобны для TDD.

Минусы:

- легко протестировать реализацию вместо поведения;
- избыток моков делает тесты хрупкими;
- не проверяют интеграцию с БД/фреймворком.

Senior short answer: unit-тест должен падать из-за изменения бизнес-контракта, а не из-за переименования приватного метода.

### Integration-Тесты

Проверяют взаимодействие нескольких компонентов: ORM + БД, repository + migrations,
queue + handler, HTTP client + fake server.

Примеры:

- сохранение агрегата в PostgreSQL;
- обработка job с реальной БД;
- проверка транзакций и блокировок;
- взаимодействие с Redis/RabbitMQ через test container или fake adapter.

Trade-off: медленнее unit, но часто дают больше уверенности для backend.

### Functional-Тесты

Проверяют функцию системы через публичный вход: HTTP endpoint, console command, message handler.

Пример: `POST /api/orders` создает заказ, публикует событие и возвращает корректный JSON.

В Laravel это часто Feature tests. В Symfony это WebTestCase/KernelTestCase.

### E2E-Тесты

Проверяют полный пользовательский сценарий максимально близко к production.

Для PHP backend обычно включают:

- реальный HTTP;
- реальную БД;
- реальные очереди или максимально близкий стенд;
- внешний UI или API gateway, если нужно.

Минусы:

- медленные;
- flaky;
- трудно дебажить;
- требуют стабильной среды.

Senior-позиция: e2e мало, но они должны защищать самые дорогие бизнес-потоки.

### Contract-Тесты

Проверяют контракт между provider и consumer: JSON schema, OpenAPI, события, очереди, HTTP API.

Используются, когда есть микросервисы, внешние клиенты, public API или event-driven integration.

Примеры контрактов:

- endpoint возвращает поле `status` только из разрешенного enum;
- event `OrderPaid` содержит `orderId`, `paidAt`, `amount`;
- provider не ломает consumer при добавлении необязательного поля.

Senior short answer: contract-тесты дешевле полного e2e между сервисами и лучше ловят breaking changes на границах.

## Test Doubles: Mocks, Stubs, Fakes, Spies

### Stub

Возвращает заранее заданные данные.

```php
$clock = new FixedClock(new DateTimeImmutable('2026-01-01 10:00:00'));
```

Используется, когда нужно контролировать входные данные: время, UUID, feature flags.

### Mock

Проверяет ожидания взаимодействия.

```php
$mailer->expects($this->once())
    ->method('send')
    ->with($this->isInstanceOf(WelcomeEmail::class));
```

Риск: mock проверяет implementation details. Если тест знает слишком много о вызовах внутри сервиса,
рефакторинг станет дорогим.

### Fake

Рабочая упрощенная реализация.

Примеры:

- in-memory repository;
- fake queue;
- fake payment gateway;
- fake event bus.

Плюс: тесты ближе к поведению, меньше завязки на конкретные вызовы.

Минус: fake может отличаться от production behavior.

### Spy

Запоминает факты вызовов, а assert делается после действия.

Пример: event dispatcher spy хранит dispatched events.

Senior short answer: stub задает ответы, mock задает ожидания, fake реализует упрощенное поведение,
spy наблюдает и проверяется после выполнения.

## Fixtures, Factories, Builders

Плохие фикстуры делают тесты хрупкими. Хорошие фикстуры минимальны и отражают смысл сценария.

Подходы:

- Object Mother: готовые объекты вроде `UserMother::admin()`.
- Test Data Builder: гибкая сборка `OrderBuilder::new()->withPaidStatus()->build()`.
- Laravel factories: быстрый способ создавать модели и состояния.
- Doctrine fixtures: полезны для базового набора данных, но не должны становиться глобальной свалкой.

Правила:

- каждый тест создает только нужные данные;
- данные должны быть читаемыми из теста;
- глобальные фикстуры использовать осторожно;
- random/faker-данные должны быть deterministic или не влиять на assert.

Антипаттерн:

```php
User::factory()->count(20)->create(); // непонятно, какие именно пользователи важны
```

Лучше:

```php
$owner = User::factory()->active()->create();
$guest = User::factory()->blocked()->create();
```

## Database Testing

Ключевые вопросы:

- использовать реальную БД или SQLite in-memory;
- очищать данные транзакциями, truncation или recreate schema;
- запускать миграции на каждый suite или переиспользовать schema snapshot;
- тестировать repository отдельно или через feature/API tests.

### Реальная БД vs SQLite

SQLite быстрее, но может скрыть ошибки:

- отличия типов;
- JSON/array columns;
- индексы;
- constraints;
- locking;
- transaction isolation;
- SQL dialect differences.

Senior short answer: для серьезной backend-системы интеграционные тесты должны запускаться на той же СУБД,
что production, например PostgreSQL/MySQL через Docker/Testcontainers/CI service.

### Transactions

Транзакции быстрые: тест оборачивается в transaction и rollback после выполнения.

Ограничения:

- не всегда работают с несколькими соединениями;
- могут скрыть проблемы commit hooks/listeners;
- плохо сочетаются с async workers, которые используют отдельное соединение;
- DDL может не откатываться одинаково в разных БД.

Альтернативы:

- truncation между тестами;
- recreate database/schema;
- отдельная база на процесс при parallel testing;
- migration snapshot.

## Очереди, События, Async Jobs

Проверять нужно два аспекта:

- dispatch: правильная job/event поставлена в очередь;
- handling: handler/job корректно выполняет бизнес-логику.

Laravel:

```php
Queue::fake();

$this->postJson('/api/orders', $payload)->assertCreated();

Queue::assertPushed(ProcessOrder::class);
```

Отдельно тестируется job:

```php
$job = new ProcessOrder($order->id);
$job->handle($processor);

$this->assertTrue($order->refresh()->processed);
```

Symfony Messenger:

- unit-тест handler'а напрямую;
- integration-тест через test transport;
- проверить envelope/stamps, retry, failure transport, idempotency.

Senior-уровень:

- async jobs должны быть идемпотентными;
- тестировать retry и dead-letter/failure queue;
- время и backoff фиксировать;
- внешние side effects изолировать;
- race conditions проверять интеграционными тестами, если бизнес-критично.

## HTTP/API Tests

Что проверять:

- status code;
- JSON schema/shape;
- business fields;
- auth/permissions;
- validation errors;
- idempotency;
- pagination/filtering/sorting;
- rate limits, если критично;
- backward compatibility для public API.

Пример Laravel:

```php
$this->actingAs($user)
    ->postJson('/api/orders', ['product_id' => $product->id])
    ->assertCreated()
    ->assertJsonPath('data.status', 'pending');
```

Пример Symfony:

```php
$client = static::createClient();
$client->request('POST', '/api/orders', [], [], ['CONTENT_TYPE' => 'application/json'], json_encode($payload));

$this->assertResponseStatusCodeSame(201);
$this->assertJsonContains(['status' => 'pending']);
```

Pitfall: не assert'ить весь JSON целиком, если контракт допускает дополнительные поля или порядок не важен.

## Legacy Code и Characterization Tests

Characterization tests фиксируют текущее поведение legacy-кода перед рефакторингом.

Цель не доказать, что поведение правильное, а защититься от случайного изменения.

Алгоритм:

1. Найти seam: публичный метод, endpoint, command, service boundary.
2. Покрыть текущее поведение через black-box тест.
3. Зафиксировать edge cases и известные баги, если они являются текущим контрактом.
4. Рефакторить малыми шагами.
5. После изменения требований обновить тесты осознанно.

Senior short answer: для legacy сначала пишу characterization tests вокруг поведения,
затем выделяю зависимости и только потом рефакторю internals.

## Flaky Tests

Причины flaky-тестов:

- реальные sleep/timeouts;
- зависимость от порядка тестов;
- shared global state;
- race conditions;
- случайные faker-данные;
- неочищенная БД/Redis/filesystem;
- внешние API;
- timezone/locale;
- parallel tests без изоляции данных.

Как лечить:

- фиксировать clock/random/UUID;
- изолировать storage per test/process;
- убрать реальные внешние сервисы из обычного CI;
- заменить sleep на polling with timeout;
- запретить зависимость от порядка;
- логировать seed и окружение;
- quarantine только временно, с задачей на исправление.

Senior short answer: flaky-тест хуже отсутствующего теста, потому что разрушает доверие к CI.

## Mutation Testing

Mutation testing проверяет качество assert'ов: инструмент вносит малые изменения в код, а тесты должны упасть.

PHP-инструмент: Infection.

Пример мутации:

```php
return $amount > 0;
```

Мутатор меняет на:

```php
return $amount >= 0;
```

Если тесты не падают, значит edge case не покрыт.

Плюсы:

- показывает слабые assert'ы;
- полезен для доменной логики;
- лучше обычного line coverage.

Минусы:

- медленный;
- требует настройки excludes;
- может быть шумным на framework glue-коде.

Senior short answer: coverage говорит, что код был выполнен, mutation score говорит, что тесты реально защищают поведение.

## Coverage

Виды покрытия:

- line coverage: строка выполнена;
- branch coverage: покрыты ветки;
- path coverage: покрыты пути;
- CRAP index: сочетание сложности и покрытия.

Что важно:

- 100% coverage не гарантирует качество;
- низкое покрытие доменной логики является риском;
- generated code, DTO, config часто можно исключать;
- покрытие должно быть порогом качества, а не самоцелью.

Практичный подход:

- высокий порог для domain/core;
- ниже или без жесткого порога для framework glue;
- diff coverage для новых изменений;
- mutation testing на критичных пакетах.

## CI Integration

Типичный pipeline:

1. Composer install/cache.
2. Static analysis: PHPStan/Psalm.
3. Coding standard: PHP-CS-Fixer/Pint/ECS.
4. Unit tests.
5. Integration/API tests с service containers.
6. Coverage report.
7. Mutation tests по расписанию или для critical modules.

Рекомендации:

- разделять быстрый suite и медленный suite;
- parallel testing включать только при изоляции БД/кэша;
- сохранять artifacts: logs, coverage, junit xml;
- не обращаться к production/staging внешним API из CI;
- использовать deterministic env: timezone, locale, seed.

## Test Data Management

Правила управления данными:

- тест сам создает свои данные;
- минимум глобального состояния;
- readable factories/states;
- explicit names для важных объектов;
- stable IDs не нужны, лучше использовать возвращенные объекты;
- cleanup должен быть автоматическим;
- PII и production dumps не использовать без анонимизации.

Для больших систем:

- seed только reference data: currencies, countries, roles;
- business data создавать в тесте;
- snapshot БД применять осторожно;
- для parallel testing нужна изоляция по database/schema/process token.

## PHPUnit

PHPUnit - базовый стандарт PHP-тестирования. Подходит для unit, integration и framework tests.

Сильные стороны:

- зрелость и экосистема;
- native assertions;
- mocks/stubs;
- data providers;
- coverage;
- интеграция с CI и IDE.

Пример:

```php
final class MoneyTest extends TestCase
{
    public function testItAddsMoneyWithSameCurrency(): void
    {
        $result = Money::eur(10)->add(Money::eur(15));

        self::assertSame(25, $result->amount());
        self::assertSame('EUR', $result->currency());
    }
}
```

Когда выбирать:

- обычные PHP-библиотеки;
- Symfony/Laravel проекты;
- команды, которым важен стандартный инструмент;
- строгий xUnit-style.

Pitfalls:

- excessive mocking;
- тестирование приватных методов;
- огромные тест-классы;
- неочевидные data providers;
- reliance on execution order.

## Codeception

Codeception - BDD-inspired framework поверх PHPUnit с удобной организацией suites: Unit, Functional, Acceptance, API.

Сильные стороны:

- удобен для acceptance/API tests;
- actor-style syntax;
- модули для Laravel, Symfony, Doctrine, REST, WebDriver;
- хорошо разделяет уровни тестирования.

Пример API-теста:

```php
$I->amBearerAuthenticated($token);
$I->sendPost('/api/orders', ['product_id' => 10]);
$I->seeResponseCodeIs(201);
$I->seeResponseContainsJson(['status' => 'pending']);
```

Когда выбирать:

- много acceptance/API/e2e сценариев;
- команда привыкла к BDD-style;
- нужен единый инструмент для REST + browser + framework modules;
- legacy-проект уже использует Codeception.

Trade-off:

- actor syntax читаемая, но может скрывать детали;
- больше инфраструктуры;
- для простых unit-тестов PHPUnit/Pest часто проще.

## Pest

Pest - современный DSL поверх PHPUnit с лаконичным синтаксисом.

Сильные стороны:

- короткий expressive syntax;
- удобен для Laravel;
- datasets;
- higher-order expectations;
- хорош для readable feature/unit tests.

Пример:

```php
it('creates an order', function () {
    $user = User::factory()->create();
    $product = Product::factory()->create();

    actingAs($user)
        ->postJson('/api/orders', ['product_id' => $product->id])
        ->assertCreated()
        ->assertJsonPath('data.status', 'pending');
});
```

Когда выбирать:

- Laravel-проекты;
- команда хочет более выразительный DSL;
- важна читаемость небольших тестов;
- уже используется PHPUnit, но хочется менее verbose syntax.

Trade-off:

- DSL нравится не всем;
- в больших enterprise-командах class-based PHPUnit может быть привычнее;
- важно не превращать closures в длинные сценарии без структуры.

## PHPUnit vs Codeception vs Pest

| Критерий | PHPUnit | Codeception | Pest |
|---|---|---|---|
| Основа | xUnit standard | поверх PHPUnit | поверх PHPUnit |
| Лучший use case | unit/integration | API/acceptance/e2e | readable unit/feature |
| Синтаксис | class-based | actor/BDD-style | closure/DSL |
| Laravel | отлично | хорошо | отлично |
| Symfony | отлично | хорошо | хорошо |
| Learning curve | низкая | средняя | низкая-средняя |
| Инфраструктура | минимальная | больше конфигурации | минимальная |

Короткий ответ на интервью: PHPUnit - база и стандарт, Pest - более выразительный DSL поверх PHPUnit,
Codeception - удобен для acceptance/API/e2e и actor-style сценариев.

## Laravel Testing Specifics

Ключевые инструменты:

- `TestCase` с application bootstrap;
- `RefreshDatabase`, `DatabaseTransactions`;
- model factories/states;
- `actingAs`, `postJson`, `assertJsonPath`;
- `Queue::fake`, `Event::fake`, `Mail::fake`, `Notification::fake`, `Bus::fake`;
- `Storage::fake`;
- HTTP client fakes;
- parallel testing.

Практические советы:

- feature tests часто ценнее изолированных controller unit tests;
- не мокать Eloquent без необходимости;
- доменную логику выносить из controller/job в сервисы и тестировать отдельно;
- проверять policies/authorization отдельно или через API сценарии;
- для jobs тестировать dispatch и handler отдельно.

Pitfall: `Event::fake()` может отключить observers/listeners, которые нужны для поведения.
Использовать scoped fake или assert после реального выполнения, если listener является частью сценария.

## Symfony Testing Specifics

Ключевые инструменты:

- `KernelTestCase` для service container/integration;
- `WebTestCase` для HTTP;
- test environment config;
- Doctrine fixtures;
- DAMA Doctrine Test Bundle или транзакционный подход;
- Messenger test transport;
- HttpClient mock responses;
- service decoration/test doubles in test container.

Практические советы:

- доменные сервисы тестировать без kernel, если возможно;
- container boot дорогой, не использовать его для чистых unit-тестов;
- repository тестировать с реальной БД;
- WebTestCase хорош для API и security flows;
- Messenger handler тестировать напрямую и через transport отдельно.

Pitfall: слишком много `KernelTestCase` делает suite медленным и маскирует плохую декомпозицию.

## Common Pitfalls и Trade-Offs

- Тестировать implementation details вместо behavior.
- Мокать ORM, query builder или framework там, где проще интеграционный тест.
- Делать один огромный e2e вместо нескольких дешевых lower-level tests.
- Использовать Faker без контроля seed и получать flaky.
- Assert'ить весь response целиком и ломать тесты при harmless changes.
- Смешивать setup/action/assert в нечитаемый сценарий.
- Полагаться только на coverage.
- Не тестировать negative paths: validation, permission denied, not found, conflict.
- Не проверять idempotency и retry для async jobs.
- Игнорировать timezone, locale, float precision, money rounding.

## Senior-Level Short Answers

**Когда mock, а когда integration test?**  
Mock использую на границе с внешним side effect или для дорогой зависимости.
Если риск в SQL/ORM/transaction behavior, нужен integration test с реальной БД.

**Почему 100% coverage недостаточно?**  
Coverage показывает выполнение строк, но не качество assert'ов.
Код может быть покрыт и при этом не проверять важный результат. Для критичной логики полезен mutation testing.

**Как тестировать очередь?**  
Отдельно проверяю, что job dispatch'ится при нужном событии, и отдельно тестирую handler/job.
Для критичных flows добавляю integration test с реальным или test transport.

**Как работать с flaky tests?**  
Сначала фиксирую причину и изоляцию: время, random, порядок, shared state, внешние сервисы.
Quarantine допустим только временно, иначе команда перестает доверять CI.

**Что важнее, unit или integration?**  
Зависит от риска. Для чистой доменной логики unit дешевле.
Для backend с ORM, транзакциями и очередями integration/API tests часто дают больше уверенности.

**Как тестировать legacy?**  
Пишу characterization tests вокруг текущего публичного поведения, затем рефакторю маленькими шагами и отделяю зависимости.

## Mini-Practice Examples

### 1. Money Rounding

Задача: протестировать расчет скидки с округлением.

Что проверить:

- обычный процент;
- граничные значения 0%, 100%;
- rounding half up/down согласно бизнес-правилу;
- разные валюты запрещены;
- отрицательная сумма невозможна.

### 2. Idempotent Job

Задача: `CapturePaymentJob` может быть выполнена дважды.

Что проверить:

- повторный запуск не создает второй payment capture;
- status не откатывается назад;
- внешний gateway вызывается один раз;
- race condition защищена lock/unique constraint.

### 3. API Contract

Задача: `GET /api/orders/{id}` для mobile app.

Что проверить:

- стабильные поля ответа;
- enum values;
- nullable поля;
- permission denied для чужого заказа;
- backward compatible addition of fields.

### 4. Legacy Service

Задача: перед рефакторингом `InvoiceCalculator`.

Что проверить:

- несколько реальных production-like сценариев;
- edge cases из баг-репортов;
- текущие странности поведения, если на них завязаны клиенты;
- после фикса бага добавить regression test.

## Self-Check Questions

- Чем unit test отличается от integration test на практике?
- Когда mock вреден?
- Почему SQLite in-memory может быть плохой заменой PostgreSQL?
- Как тестировать транзакции и rollback behavior?
- Как проверить, что job и dispatch'ится, и корректно выполняется?
- Какой тест написать перед рефакторингом legacy-кода?
- Что делать с flaky-тестом, который падает раз в неделю?
- Что показывает mutation testing, чего не показывает line coverage?
- Какие данные должны быть глобальными fixtures, а какие создаваться в тесте?
- Как разделить быстрый и медленный test suite в CI?
- Когда выбрать Codeception вместо PHPUnit/Pest?
- Какие особенности Laravel/Symfony testing чаще всего влияют на скорость suite?

## Code Review Checklist для Тестов

- Тест проверяет поведение, а не внутреннюю реализацию.
- Название теста объясняет сценарий и ожидаемый результат.
- Setup минимальный и читаемый.
- Arrange/Act/Assert визуально разделены.
- Нет зависимости от порядка выполнения тестов.
- Нет реальных внешних API в обычном suite.
- Время, random, UUID и timezone контролируются.
- БД/Redis/filesystem очищаются или изолируются.
- Проверены positive и critical negative paths.
- Assert'ы достаточно специфичны, но не хрупкие.
- Моки используются только на разумных границах.
- Для async проверены dispatch, handler, retry/idempotency при необходимости.
- Для API проверены status, auth, validation и contract-important fields.
- Тест не слишком медленный для своего уровня.
- Новая логика покрыта на подходящем уровне пирамиды.
- Если есть bugfix, добавлен regression test.

## Что Говорить на Интервью

Хороший ответ Senior/Lead должен звучать так:

> Я начинаю не с выбора PHPUnit/Pest/Codeception, а с карты рисков.
> Где бизнес-логика, где интеграционные границы, где высокая цена регрессии.
> Unit-тестами закрываю чистую логику, integration/API тестами - БД, транзакции, очереди и фреймворк,
> e2e оставляю для критичных flows. Следую принципу: тест должен быть быстрым, надежным,
> читаемым и падать по полезной причине.

## Ссылки

Официальная документация:

- PHPUnit: <https://phpunit.de/>
- PHPUnit Documentation: <https://docs.phpunit.de/>
- Codeception: <https://codeception.com/>
- Codeception Documentation: <https://codeception.com/docs/>
- Pest: <https://pestphp.com/>
- Pest Documentation: <https://pestphp.com/docs>
- Laravel Testing: <https://laravel.com/docs/testing>
- Laravel HTTP Tests: <https://laravel.com/docs/http-tests>
- Laravel Mocking/Fakes: <https://laravel.com/docs/mocking>
- Symfony Testing: <https://symfony.com/doc/current/testing.html>
- Symfony Messenger Testing: <https://symfony.com/doc/current/messenger.html#testing>

Статьи и материалы:

- Martin Fowler, Test Pyramid: <https://martinfowler.com/bliki/TestPyramid.html>
- Martin Fowler, Mocks Aren't Stubs: <https://martinfowler.com/articles/mocksArentStubs.html>
- Google Testing Blog: <https://testing.googleblog.com/>
- Infection Mutation Testing: <https://infection.github.io/>
- Infection Documentation: <https://infection.github.io/guide/>
- Microsoft Azure Architecture Center, Retry pattern: <https://learn.microsoft.com/en-us/azure/architecture/patterns/retry>
- Microservices.io, Transactional Outbox: <https://microservices.io/patterns/data/transactional-outbox.html>
- Stripe API idempotent requests: <https://docs.stripe.com/api/idempotent_requests>
