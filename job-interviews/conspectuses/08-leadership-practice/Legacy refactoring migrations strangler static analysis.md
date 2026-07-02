# Legacy Refactoring, Migrations, Strangler Fig, Static Analysis

Гайд для Lead/Senior PHP backend developer: как говорить на интервью о работе с legacy-системами, безопасных миграциях, постепенной модернизации и снижении технического риска.

## Как подходить к legacy-системе

Legacy-код - это не просто старый код. Практичное определение: код, который сложно безопасно менять, потому что поведение плохо описано тестами, архитектурой, документацией или наблюдаемостью.

На интервью важно показывать не желание "переписать все", а умение управлять риском.

Рабочий подход:

1. Понять бизнес-критичность модулей: деньги, заказы, платежи, авторизация, интеграции.
2. Собрать карту системы: входные точки, БД, очереди, cron, внешние API, shared state.
3. Включить наблюдаемость до изменений: логи, метрики, трассировка, алерты, business KPIs.
4. Зафиксировать текущее поведение characterization tests.
5. Делать маленькие изменения с быстрым rollback.
6. Уменьшать coupling через seams, интерфейсы, адаптеры, dependency inversion.
7. Постепенно повышать качество: static analysis, типы, Rector, тесты, modularization.

Senior-level формулировка:

> Я не начинаю с переписывания. Сначала стабилизирую контур изменений: наблюдаемость, characterization tests, карта рисков, маленькие reversible changes. Потом выделяю seams и постепенно заменяю части системы, сохраняя поведение.

## Characterization Tests

Characterization tests фиксируют текущее поведение системы, даже если оно выглядит странным. Их цель - не доказать правильность кода, а защитить от случайных регрессий.

Когда нужны:

- код без тестов;
- непонятные business rules;
- сложные edge cases;
- перед рефакторингом или миграцией;
- перед заменой интеграции, ORM, фреймворка, версии PHP.

Пример PHPUnit для legacy-сервиса:

```php
final class PriceCalculatorCharacterizationTest extends TestCase
{
    public function testCalculatesDiscountForVipCustomer(): void
    {
        $calculator = new LegacyPriceCalculator();

        $result = $calculator->calculate(
            amount: 10000,
            customerType: 'vip',
            coupon: 'SUMMER'
        );

        self::assertSame(8200, $result);
    }
}
```

В Laravel можно зафиксировать поведение HTTP endpoint:

```php
public function testLegacyCheckoutResponseShape(): void
{
    Http::fake([
        'payment-gateway.local/*' => Http::response(['status' => 'ok']),
    ]);

    $response = $this->postJson('/api/checkout', [
        'cart_id' => 123,
        'coupon' => 'SUMMER',
    ]);

    $response->assertOk()
        ->assertJsonPath('status', 'paid')
        ->assertJsonStructure(['order_id', 'status', 'total']);
}
```

Практический совет: начинай с тестов на публичные интерфейсы: HTTP, console commands, queues, events, repository/service API. Не тестируй сразу приватные детали.

## Работа с кодом без тестов

Если тестов нет, не нужно пытаться покрыть все сразу.

Минимальный безопасный порядок:

1. Добавить smoke tests для самых критичных сценариев.
2. Добавить characterization tests вокруг изменяемого участка.
3. Зафиксировать внешние контракты: JSON response, events, DB side effects, external calls.
4. Изолировать внешние зависимости fake/mock/stub-ами.
5. Делать refactor только после зеленых тестов.

Если код трудно тестировать:

- используй seams: выдели место, где можно подменить зависимость;
- оберни static/facade/global вызовы адаптером;
- добавь интерфейс вокруг внешней интеграции;
- раздели pure business logic и infrastructure code;
- сначала меняй структуру без изменения поведения.

Пример seam вокруг статического вызова:

```php
interface Clock
{
    public function now(): DateTimeImmutable;
}

final class SystemClock implements Clock
{
    public function now(): DateTimeImmutable
    {
        return new DateTimeImmutable();
    }
}
```

Теперь код можно тестировать без зависимости от реального времени.

## Safe Refactoring

Refactoring - изменение структуры без изменения внешнего поведения.

Безопасные техники:

- extract method/class;
- introduce parameter object;
- replace conditional with polymorphism, если условная логика действительно растет;
- move method ближе к данным;
- replace static/global dependency with injected dependency;
- split query and command;
- introduce adapter/facade вокруг внешних систем;
- parallel run старой и новой реализации;
- branch by abstraction.

Правило: не смешивать refactoring и feature change в одном PR, если это можно разделить.

Пример плохого PR:

- обновили PHP;
- переписали checkout;
- поменяли схему БД;
- добавили новую бизнес-логику;
- удалили старый код.

Пример хорошего порядка:

1. Добавили тесты и метрики.
2. Выделили интерфейс.
3. Подключили новую реализацию за feature flag.
4. Запустили shadow/parallel mode.
5. Переключили малый процент трафика.
6. Удалили старую реализацию после стабилизации.

## Seams и Dependency Inversion

Seam - точка, где можно изменить поведение без массового изменения кода. В PHP legacy seams часто создают вокруг:

- static helpers;
- Laravel Facades;
- Doctrine EntityManager;
- Eloquent models;
- внешних API clients;
- файловой системы;
- времени;
- глобальных конфигов;
- mail/sms/payment providers.

Dependency Inversion означает, что бизнес-логика зависит от абстракции, а инфраструктура реализует эту абстракцию.

Symfony-пример:

```php
interface PaymentGateway
{
    public function charge(Money $amount, CustomerId $customerId): PaymentResult;
}

final class StripePaymentGateway implements PaymentGateway
{
    public function charge(Money $amount, CustomerId $customerId): PaymentResult
    {
        // Stripe SDK call
    }
}

final class CheckoutService
{
    public function __construct(private PaymentGateway $payments) {}

    public function checkout(Order $order): PaymentResult
    {
        return $this->payments->charge($order->total(), $order->customerId());
    }
}
```

Laravel binding:

```php
$this->app->bind(PaymentGateway::class, StripePaymentGateway::class);
```

Tradeoff: абстракции полезны вокруг нестабильных или внешних зависимостей. Не нужно вводить интерфейс для каждого класса автоматически.

## Static Analysis: PHPStan и Psalm

Static analysis помогает находить ошибки до runtime и постепенно повышать строгость кода.

Практический adoption plan:

1. Установить PHPStan или Psalm.
2. Запустить на минимальном уровне.
3. Создать baseline для существующих ошибок.
4. Запретить новые ошибки в CI.
5. Постепенно повышать level.
6. Уменьшать baseline по модулю или типу ошибок.
7. Добавить generics/phpdoc там, где PHP-типы недостаточны.

PHPStan:

```bash
vendor/bin/phpstan analyse --level=0 src
vendor/bin/phpstan analyse --generate-baseline
```

Psalm:

```bash
vendor/bin/psalm --init
vendor/bin/psalm --set-baseline=psalm-baseline.xml
```

Пример PHPDoc generics для коллекции:

```php
/**
 * @return list<User>
 */
public function activeUsers(): array
{
    return $this->repository->findActive();
}
```

Senior-level ответ:

> Я обычно внедряю static analysis через baseline, чтобы не блокировать команду тысячами старых ошибок. В CI запрещаю новые ошибки, а baseline уменьшаю итеративно. Уровень повышаю после стабилизации, начиная с критичных доменных модулей.

Pitfalls:

- включить высокий level сразу и парализовать delivery;
- бесконтрольно добавлять ignoreErrors;
- воспринимать baseline как мусорную корзину навсегда;
- заменять тесты static analysis-ом;
- добавлять ложные типы в PHPDoc только ради прохождения анализа.

## Rector Migrations

Rector автоматизирует mechanical refactoring: обновление синтаксиса, deprecated API, type declarations, framework rules.

Где полезен:

- PHP version upgrade;
- Symfony/Laravel upgrade;
- добавление type declarations;
- замена deprecated вызовов;
- унификация code style;
- массовые безопасные преобразования.

Пример rector.php:

```php
use Rector\Config\RectorConfig;
use Rector\Set\ValueObject\LevelSetList;

return RectorConfig::configure()
    ->withPaths([__DIR__ . '/src'])
    ->withPhpSets(php82: true)
    ->withSets([
        LevelSetList::UP_TO_PHP_82,
    ]);
```

Команды:

```bash
vendor/bin/rector process --dry-run
vendor/bin/rector process
```

Практика:

- запускать Rector маленькими наборами rules;
- делать отдельные PR для mechanical changes;
- проверять diff глазами;
- после Rector гонять tests, static analysis, smoke tests;
- не смешивать automated refactor с бизнес-изменениями.

Tradeoff: Rector ускоряет массовые изменения, но не понимает весь business context. Нельзя слепо принимать diff на критичных участках.

## PHP Version Upgrades

Типичный порядок upgrade PHP 7.4 -> 8.x:

1. Проверить поддержку dependencies.
2. Обновить composer constraints.
3. Запустить tests и static analysis на текущей версии.
4. Прогнать Rector для целевой версии.
5. Исправить breaking changes.
6. Проверить runtime warnings/deprecations.
7. Обновить Docker/CI/FPM/CLI окружение.
8. Выкатить canary/staging.
9. Следить за errors, latency, memory, queue failures.

Частые проблемы PHP 8.x:

- более строгие TypeError/ValueError;
- named arguments ломают вызовы при изменении names;
- изменения в internal functions;
- dynamic properties deprecated с PHP 8.2;
- несовместимые composer packages;
- различия CLI/FPM extensions.

Senior-level ответ:

> PHP upgrade я рассматриваю как production migration: сначала dependency audit и CI matrix, затем Rector и deprecation fixes, потом staging/canary. Важно обновить не только код, но и runtime: Docker images, FPM, extensions, opcache settings, CI.

## Framework Upgrades: Laravel и Symfony

Framework upgrade лучше делать последовательно по major/minor версиям, не перепрыгивая через несколько major без необходимости.

Laravel checklist:

- прочитать official upgrade guide для каждой версии;
- обновить composer constraints;
- проверить breaking changes в config, middleware, validation, queues, filesystem, auth;
- обновить first-party packages: Horizon, Sanctum, Passport, Cashier, Scout;
- проверить кастомные service providers;
- проверить route model binding и policies;
- прогнать feature tests и queue jobs.

Symfony checklist:

- включить deprecation notices;
- обновлять компоненты постепенно;
- проверить recipes/flex changes;
- проверить DI container autowiring/autoconfiguration;
- обновить security config, annotations/attributes, event subscribers;
- использовать Symfony PHPUnit Bridge для deprecations;
- удалить deprecated API до перехода на следующий major.

Pitfalls:

- игнорировать deprecations до major upgrade;
- обновлять framework вместе с большими feature changes;
- забыть queue workers, scheduler, consumers;
- проверить только HTTP, но не CLI/cron/background jobs.

## Database Migrations

DB migration в production должна учитывать locks, volume, replication lag, backward compatibility и rollback.

Безопасный expand/contract pattern:

1. Expand: добавить новую колонку/таблицу nullable или с безопасным default.
2. Deploy code, который пишет и в старую, и в новую структуру.
3. Backfill данных batch-ами.
4. Read switch: начать читать из новой структуры.
5. Проверить consistency.
6. Contract: удалить старую колонку/таблицу после периода стабилизации.

Пример Laravel migration:

```php
Schema::table('orders', function (Blueprint $table): void {
    $table->string('external_payment_id')->nullable()->index();
});
```

Для больших таблиц:

- не добавлять тяжелые non-null default без понимания СУБД;
- делать backfill отдельным job/command batch-ами;
- использовать online schema change tools, если нужны;
- проверять locks на staging с похожим объемом;
- иметь rollback plan.

Пример backfill batch-ами:

```php
Order::query()
    ->whereNull('external_payment_id')
    ->orderBy('id')
    ->chunkById(1000, function ($orders): void {
        foreach ($orders as $order) {
            $order->updateQuietly([
                'external_payment_id' => $order->legacy_payment_ref,
            ]);
        }
    });
```

Tradeoff: rollback DB migration часто сложнее rollback кода. Поэтому schema changes должны быть backward-compatible.

## Strangler Fig Pattern

Strangler Fig - постепенная замена legacy-системы новой реализацией, когда новый код оборачивает старый и перехватывает все больше сценариев.

Когда подходит:

- большой монолит нельзя переписать сразу;
- есть clear bounded contexts;
- можно маршрутизировать часть трафика;
- есть стабильные external contracts;
- нужно снижать риск миграции.

Типовая схема:

1. Поставить proxy/router/facade перед legacy.
2. Выбрать один bounded context или use case.
3. Реализовать новый сервис/модуль.
4. Направить часть запросов в новую реализацию.
5. Сравнивать результаты через shadow traffic/parallel run.
6. Постепенно расширять покрытие.
7. Удалить старую часть, когда она больше не используется.

Laravel route-level пример:

```php
Route::post('/checkout', function (Request $request, CheckoutRouter $router) {
    return $router->handle($request);
});
```

```php
final class CheckoutRouter
{
    public function __construct(
        private LegacyCheckout $legacy,
        private NewCheckout $modern,
        private FeatureFlags $flags,
    ) {}

    public function handle(Request $request): JsonResponse
    {
        if ($this->flags->enabled('new_checkout', $request->user())) {
            return $this->modern->handle($request);
        }

        return $this->legacy->handle($request);
    }
}
```

Pitfalls:

- слишком долго держать две системы;
- не иметь плана удаления legacy;
- shared database становится hidden coupling;
- разные модели данных приводят к consistency bugs;
- нет observability для сравнения старого и нового поведения.

Senior-level ответ:

> Strangler Fig я применяю, когда rewrite слишком рискован. Сначала ставлю routing/facade слой, затем переношу один bounded context, включаю через feature flags, сравниваю поведение и постепенно отрезаю legacy. Критично иметь критерии завершения, иначе получится вечная двойная поддержка.

## Модульность и Extracting Composer Packages/Services

Перед физическим extraction лучше добиться логической модульности внутри монолита.

Порядок:

1. Определить bounded context.
2. Запретить прямые зависимости между модулями.
3. Ввести public API модуля: application services, commands, queries, events.
4. Убрать shared mutable state.
5. Разделить migrations, configs, routes, tests.
6. Только потом выделять composer package или сервис.

Composer package подходит, если:

- код переиспользуется в нескольких приложениях;
- есть стабильный API;
- зависимости ограничены;
- можно версионировать и тестировать отдельно.

Отдельный сервис подходит, если:

- нужна независимая масштабируемость;
- разный release cycle;
- сильная доменная граница;
- команда готова поддерживать network, observability, deploy, contracts.

Tradeoff:

- package проще, чем microservice;
- service добавляет network failures, latency, distributed transactions;
- преждевременное выделение сервиса часто ухудшает систему.

Senior-level ответ:

> Я не начинаю с микросервиса. Сначала делаю модуль внутри монолита с четким API и тестами. Если граница стабильна и есть operational reason, тогда выношу в package или сервис.

## Observability Before Refactor

До рефакторинга нужно понимать текущее production-поведение.

Что добавить:

- structured logs с correlation/request id;
- error rate по endpoint/job/consumer;
- latency percentiles p50/p95/p99;
- business metrics: orders created, payments failed, refunds, conversion;
- queue lag, retry count, dead letters;
- external API latency/error rate;
- database slow queries, locks, replication lag;
- dashboards и alerts.

Пример Laravel logging context:

```php
Log::withContext([
    'request_id' => $request->headers->get('X-Request-Id'),
    'user_id' => $request->user()?->id,
]);
```

Практическая фраза:

> Если я не могу измерить поведение до изменения, я не смогу доказать, что после изменения стало безопасно.

## Feature Flags

Feature flags позволяют отделить deploy от release.

Применения:

- включить новую реализацию для команды;
- canary rollout на 1%, 5%, 25%, 100%;
- быстро выключить проблемную ветку;
- A/B или shadow mode;
- миграция клиентов/тенантов пачками.

Важно:

- flags должны иметь owner и срок удаления;
- не держать временные flags годами;
- логировать значение flag в request context;
- тестировать обе ветки, пока они активны;
- избегать взрыва комбинаций flags.

## Risk Management и Rollback

Риск legacy-изменений управляется не героизмом, а процессом.

Checklist:

- маленький PR;
- clear acceptance criteria;
- тесты вокруг изменяемого поведения;
- backward-compatible schema changes;
- feature flag;
- canary/staged rollout;
- метрики и алерты;
- rollback plan;
- data recovery plan;
- communication plan для support/product.

Rollback бывает разным:

- code rollback;
- выключить feature flag;
- откатить routing;
- остановить backfill;
- восстановить данные из backup/event log;
- forward fix, если DB rollback опаснее.

Senior-level ответ:

> Для меня rollback plan - часть дизайна изменения. Особенно при DB migration я предпочитаю backward-compatible expand/contract, чтобы можно было откатить код без немедленного отката схемы.

## Technical Debt Prioritization

Не весь technical debt одинаково важен. Приоритизация должна быть связана с business impact и engineering throughput.

Критерии:

- частота изменений в модуле;
- количество инцидентов;
- влияние на revenue/security/compliance;
- сложность onboarding;
- coupling с другими частями;
- отсутствие тестов в критичной зоне;
- blocker для продуктовых инициатив;
- cost of delay.

Подходы:

- Boy Scout Rule: улучшать код рядом с текущей задачей;
- debt budget: фиксированный процент capacity;
- risk-based roadmap;
- cleanup после миграции;
- явно удалять dead code и старые flags.

Pitfall: продавать технический долг как "разработчикам не нравится код". Лучше говорить: "этот долг увеличивает lead time, вызывает инциденты и блокирует upgrade".

## Common Pitfalls и Tradeoffs

Частые ошибки:

- Big Bang rewrite без incremental delivery;
- начать с архитектуры, не поняв production behavior;
- нет тестов вокруг изменяемого участка;
- нет feature flags и rollback;
- смешать refactoring, migration и feature в одном релизе;
- игнорировать data migration complexity;
- слишком много абстракций заранее;
- baseline static analysis становится постоянной свалкой;
- не удалить старый код после Strangler migration;
- недооценить queue workers, cron, CLI commands.

Tradeoffs:

- baseline ускоряет внедрение static analysis, но требует дисциплины удаления;
- feature flags снижают release risk, но увеличивают сложность тестирования;
- monolith modularization проще operationally, но не дает независимого scaling;
- microservice дает независимость, но добавляет distributed systems complexity;
- Rector ускоряет mechanical changes, но требует review;
- characterization tests могут зафиксировать баги, но защищают от случайных изменений поведения.

## Senior-Level Short Answers

### Как рефакторить legacy без тестов?

Сначала добавляю characterization tests вокруг публичного поведения и observability. Затем выделяю seams, делаю маленькие refactoring steps без изменения поведения, проверяю тестами и метриками. Не смешиваю рефакторинг с новой функциональностью.

### Когда выбирать rewrite, а когда Strangler Fig?

Rewrite оправдан редко: если система маленькая, домен понятен, данных мало, интеграций почти нет. Для критичного большого legacy чаще выбираю Strangler Fig: постепенно переношу bounded contexts, включаю через routing/flags, сравниваю поведение и удаляю старый код поэтапно.

### Как внедрить PHPStan в большой legacy-проект?

Запускаю на низком level, создаю baseline, включаю CI так, чтобы новые ошибки не проходили. Потом уменьшаю baseline по модулям и повышаю level. Важно не превращать ignoreErrors в постоянную норму.

### Как безопасно делать DB migration?

Использую expand/contract: сначала backward-compatible schema, потом dual-write/backfill/read-switch, потом удаление старой структуры. Для больших таблиц избегаю долгих locks и делаю backfill batch-ами.

### Как обновлять PHP или framework?

Сначала dependency audit и deprecation cleanup, затем Rector/static analysis/tests, потом CI/runtime upgrade, staging/canary и observability. Не совмещаю upgrade с крупными feature changes.

### Как объяснить technical debt бизнесу?

Через риск, стоимость и скорость: долг увеличивает lead time, повышает вероятность инцидентов, блокирует upgrades и продуктовые изменения. Приоритизирую долг там, где он влияет на revenue, reliability или delivery.

## Mini-Practice

### Практика 1: legacy endpoint без тестов

Задача: нужно изменить расчет скидки в `/checkout`.

План:

1. Найти все входы: HTTP, jobs, admin, API clients.
2. Добавить characterization feature tests на текущие сценарии.
3. Зафиксировать external calls через fake/mock.
4. Добавить business metrics: checkout success/fail, discount amount.
5. Выделить `DiscountCalculator` за интерфейсом.
6. Включить новую реализацию через feature flag.
7. Сравнить результаты старой и новой логики на shadow mode.

### Практика 2: PHPStan adoption checklist

1. Установлен PHPStan/Psalm.
2. Есть baseline.
3. CI запрещает новые ошибки.
4. Есть owner baseline-файла.
5. Есть план уменьшения baseline.
6. Новые модули пишутся на более строгом уровне.
7. Ignore rules имеют комментарий или issue.

### Практика 3: DB migration checklist

1. Миграция backward-compatible.
2. Проверены locks и время выполнения.
3. Есть backfill strategy.
4. Есть consistency checks.
5. Код может работать со старой и новой схемой.
6. Rollback кода безопасен.
7. Удаление старой схемы вынесено в отдельный релиз.

### Практика 4: Strangler checklist

1. Выбран bounded context.
2. Есть routing/facade слой.
3. Есть feature flag или tenant-based rollout.
4. Есть метрики сравнения старой и новой реализации.
5. Есть план синхронизации данных.
6. Есть критерии удаления legacy.
7. Старый код удаляется после стабилизации.

## Self-Check Questions

1. Чем legacy-код отличается от просто старого кода?
2. Зачем нужны characterization tests и чем они отличаются от обычных unit tests?
3. Что такое seam и как его создать в PHP legacy-коде?
4. Почему нельзя смешивать refactoring и feature change?
5. Как внедрить PHPStan/Psalm без остановки разработки?
6. Какие риски есть у Rector migrations?
7. Как безопасно обновлять PHP major version?
8. Что проверить при Laravel/Symfony upgrade кроме HTTP endpoints?
9. Как работает expand/contract для DB migrations?
10. Когда Strangler Fig лучше Big Bang rewrite?
11. Какие риски дает shared database при выделении сервиса?
12. Почему observability нужно добавить до рефакторинга?
13. Как управлять feature flags, чтобы они не стали technical debt?
14. Какие варианты rollback существуют кроме `git revert`?
15. Как приоритизировать technical debt перед бизнесом?

## Interview Checklist

Перед интервью умей объяснить:

1. Как ты заходишь в незнакомый legacy-проект.
2. Как защищаешь поведение перед refactoring.
3. Как работаешь с кодом без тестов.
4. Как внедряешь static analysis постепенно.
5. Как используешь Rector и где ему не доверяешь.
6. Как планируешь PHP/framework upgrade.
7. Как делаешь безопасные DB migrations.
8. Как применяешь Strangler Fig.
9. Как решаешь, выносить ли код в package/service.
10. Как управляешь rollout, rollback и observability.

## Ссылки

- Martin Fowler: Strangler Fig Application - https://martinfowler.com/bliki/StranglerFigApplication.html
- Martin Fowler: Refactoring - https://martinfowler.com/books/refactoring.html
- Martin Fowler: Branch By Abstraction - https://martinfowler.com/bliki/BranchByAbstraction.html
- Michael Feathers: Working Effectively with Legacy Code - https://www.oreilly.com/library/view/working-effectively-with/0131177052/
- PHPStan documentation - https://phpstan.org/user-guide/getting-started
- PHPStan baseline - https://phpstan.org/user-guide/baseline
- Psalm documentation - https://psalm.dev/docs/
- Psalm issue baseline - https://psalm.dev/docs/running_psalm/dealing_with_code_issues/#using-a-baseline-file
- Rector documentation - https://getrector.com/documentation
- Rector PHP upgrades - https://getrector.com/documentation/set-lists
- Laravel upgrade guide - https://laravel.com/docs/upgrade
- Symfony upgrade guide - https://symfony.com/doc/current/setup/upgrade_major.html
- Symfony deprecations - https://symfony.com/doc/current/setup/upgrade_major.html#deprecations-in-phpunit
