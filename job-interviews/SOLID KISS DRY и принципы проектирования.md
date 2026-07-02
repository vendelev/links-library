# SOLID, KISS, DRY и принципы проектирования

Практический конспект для Lead/Senior PHP-разработчика перед интервью. Фокус: как объяснять принципы, где они помогают, где вредят, и как применять их в PHP-коде без догматизма.

## Как говорить на интервью

Хороший senior-level ответ обычно содержит:

- краткое определение принципа;
- проблему, которую он решает;
- пример из реального кода;
- компромисс или типичное злоупотребление;
- связь с тестируемостью, изменяемостью и стоимостью сопровождения.

Плохой ответ: пересказать аббревиатуру SOLID без контекста. Хороший ответ: показать, как принцип снижает риск изменения бизнес-логики, инфраструктуры или интеграций.

## SOLID

SOLID - набор принципов объектно-ориентированного проектирования, популяризированный Robert C. Martin. Это не чеклист синтаксиса, а эвристики для управления зависимостями и изменениями.

### SRP - Single Responsibility Principle

Класс должен иметь одну причину для изменения. Ответственность - это не "один метод" и не "делать только одну операцию", а одна ось изменений.

Антипример:

```php
final class InvoiceService
{
    public function create(array $data): Invoice
    {
        $invoice = Invoice::create($data);

        $pdf = $this->renderPdf($invoice);
        file_put_contents('/tmp/invoice.pdf', $pdf);

        mail($invoice->customer_email, 'Invoice', 'Attached');

        return $invoice;
    }

    private function renderPdf(Invoice $invoice): string
    {
        return '...pdf bytes...';
    }
}
```

Проблема: создание счета, генерация PDF, хранение файла и отправка письма меняются по разным причинам.

Лучше:

```php
final class InvoiceCreator
{
    public function __construct(
        private InvoiceRepository $invoices,
        private InvoicePdfGenerator $pdfGenerator,
        private InvoiceStorage $storage,
        private InvoiceNotifier $notifier,
    ) {}

    public function create(CreateInvoiceData $data): Invoice
    {
        $invoice = $this->invoices->save(Invoice::fromData($data));
        $path = $this->storage->store($invoice, $this->pdfGenerator->generate($invoice));
        $this->notifier->invoiceCreated($invoice, $path);

        return $invoice;
    }
}
```

Senior-level ответ: SRP помогает локализовать изменения. Но дробить каждый метод в отдельный класс не нужно: слишком мелкая декомпозиция ухудшает навигацию и повышает когнитивную нагрузку.

Типичные ошибки:

- путать SRP с "класс должен иметь один метод";
- создавать `Manager`, `Helper`, `Service`, которые делают все;
- преждевременно разносить простой use case на десятки классов.

### OCP - Open/Closed Principle

Модуль должен быть открыт для расширения и закрыт для изменения. Идея: новое поведение добавляется без переписывания стабильного кода.

Антипример:

```php
final class DiscountCalculator
{
    public function calculate(Order $order): Money
    {
        return match ($order->customerType()) {
            'regular' => $order->total()->multiply(0.05),
            'vip' => $order->total()->multiply(0.15),
            'partner' => $order->total()->multiply(0.20),
            default => Money::zero(),
        };
    }
}
```

Если типы скидок часто добавляются, `match` будет постоянно меняться.

Вариант через стратегии:

```php
interface DiscountPolicy
{
    public function supports(Order $order): bool;
    public function discountFor(Order $order): Money;
}

final class VipDiscountPolicy implements DiscountPolicy
{
    public function supports(Order $order): bool
    {
        return $order->customerType() === CustomerType::Vip;
    }

    public function discountFor(Order $order): Money
    {
        return $order->total()->multiply(0.15);
    }
}

final class DiscountCalculator
{
    /** @param iterable<DiscountPolicy> $policies */
    public function __construct(private iterable $policies) {}

    public function calculate(Order $order): Money
    {
        foreach ($this->policies as $policy) {
            if ($policy->supports($order)) {
                return $policy->discountFor($order);
            }
        }

        return Money::zero();
    }
}
```

Senior-level ответ: OCP оправдан, когда есть стабильная точка расширения и реально ожидаемая вариативность: платежные провайдеры, способы доставки, правила ценообразования. Для одноразовой ветки `if` стратегия может быть overengineering.

Tradeoff: OCP против KISS. Не нужно строить plugin architecture там, где изменение случится один раз и дешевле изменить код напрямую.

### LSP - Liskov Substitution Principle

Подтип должен быть взаимозаменяем с базовым типом без нарушения корректности программы. Наследник не должен усиливать предусловия, ослаблять постусловия или неожиданно менять семантику.

Антипример:

```php
class FileStorage
{
    public function save(string $path, string $contents): void
    {
        file_put_contents($path, $contents);
    }
}

final class ReadOnlyStorage extends FileStorage
{
    public function save(string $path, string $contents): void
    {
        throw new RuntimeException('Read-only storage');
    }
}
```

Код, ожидающий `FileStorage`, не может безопасно использовать `ReadOnlyStorage`.

Лучше разделить контракты:

```php
interface FileReader
{
    public function read(string $path): string;
}

interface FileWriter
{
    public function save(string $path, string $contents): void;
}

final class LocalFileStorage implements FileReader, FileWriter
{
    public function read(string $path): string
    {
        return file_get_contents($path) ?: '';
    }

    public function save(string $path, string $contents): void
    {
        file_put_contents($path, $contents);
    }
}
```

Senior-level ответ: LSP часто проявляется не в синтаксисе, а в контрактах: исключения, nullable-результаты, побочные эффекты, единицы измерения, порядок вызовов, транзакционные гарантии.

Типичные ошибки:

- наследник кидает `NotSupportedException` для метода родителя;
- наследник требует более строгий формат входных данных;
- наследник возвращает `null`, хотя базовый контракт обещает объект;
- наследование используется ради переиспользования кода, а не ради отношения "is-a".

### ISP - Interface Segregation Principle

Клиенты не должны зависеть от методов, которые они не используют. Лучше несколько узких интерфейсов, чем один жирный.

Антипример:

```php
interface UserRepository
{
    public function findById(UserId $id): ?User;
    public function save(User $user): void;
    public function delete(UserId $id): void;
    public function exportCsv(): string;
    public function syncToCrm(User $user): void;
}
```

Лучше:

```php
interface UserProvider
{
    public function findById(UserId $id): ?User;
}

interface UserPersister
{
    public function save(User $user): void;
}

interface UserExporter
{
    public function exportCsv(): string;
}
```

Senior-level ответ: ISP уменьшает лишние зависимости и упрощает тестовые doubles. Но не надо создавать интерфейс для каждого класса автоматически. Интерфейс полезен, когда есть несколько реализаций, граница слоя, контракт для внешнего клиента или нужно изолировать инфраструктуру.

### DIP - Dependency Inversion Principle

Высокоуровневые модули не должны зависеть от низкоуровневых деталей. Оба должны зависеть от абстракций. Абстракции не должны зависеть от деталей.

Антипример:

```php
final class RegisterUserHandler
{
    public function handle(RegisterUserCommand $command): void
    {
        $user = new User($command->email);

        $pdo = new PDO($_ENV['DSN']);
        $pdo->prepare('INSERT INTO users ...')->execute([...]);

        (new SmtpMailer())->send($user->email(), 'Welcome');
    }
}
```

Лучше:

```php
final class RegisterUserHandler
{
    public function __construct(
        private UserRepository $users,
        private WelcomeMailer $mailer,
    ) {}

    public function handle(RegisterUserCommand $command): void
    {
        $user = User::register(Email::fromString($command->email));

        $this->users->save($user);
        $this->mailer->sendTo($user);
    }
}
```

Senior-level ответ: DIP - не про "везде интерфейсы", а про направление зависимостей. Доменная логика не должна знать про PDO, HTTP-клиент, Redis, SMTP или Laravel Facade, если это затрудняет тестирование и замену деталей.

В PHP-проектах DIP часто реализуется через:

- constructor injection;
- контейнер Symfony/Laravel;
- интерфейсы на границах приложения;
- адаптеры для внешних сервисов;
- порты и адаптеры в hexagonal architecture.

## KISS - Keep It Simple, Stupid

Выбирай самое простое решение, которое корректно закрывает текущую задачу и понятно команде.

Антипример overengineering:

```php
interface ActiveUserSpecificationFactoryInterface
{
    public function create(): SpecificationInterface;
}
```

Если вся задача - проверить активность пользователя, достаточно:

```php
if (!$user->isActive()) {
    throw new UserIsNotActive();
}
```

Senior-level ответ: KISS не означает "писать примитивно". Простота - это низкая случайная сложность: меньше ненужных слоев, меньше магии, понятные зависимости, явные инварианты.

Практические признаки нарушения KISS:

- паттерн добавлен только потому, что "так правильно";
- невозможно понять flow без IDE и дебаггера;
- абстракция имеет одну реализацию и не защищает важную границу;
- простая бизнес-операция проходит через 8 классов без причины.

## DRY - Don't Repeat Yourself

Каждое знание должно иметь единственное, однозначное представление в системе. DRY не про удаление всех похожих строк, а про устранение дублирования знания.

Плохое дублирование:

```php
final class OrderValidator
{
    public function validate(Order $order): void
    {
        if ($order->total()->lessThan(Money::of(100))) {
            throw new DomainException('Minimum order total is 100');
        }
    }
}

final class CheckoutController
{
    public function __invoke(Request $request): Response
    {
        if ($request->integer('total') < 100) {
            return new JsonResponse(['error' => 'Minimum order total is 100'], 422);
        }
    }
}
```

Правило минимальной суммы продублировано в домене и контроллере.

Лучше держать правило в домене, а контроллеру отображать ошибку:

```php
final class Order
{
    private const MIN_TOTAL = 100;

    public function assertCanBeCheckedOut(): void
    {
        if ($this->total->lessThan(Money::of(self::MIN_TOTAL))) {
            throw new MinimumOrderTotalViolation(self::MIN_TOTAL);
        }
    }
}
```

Опасный DRY:

```php
final class UserAndProductStatusHelper
{
    public static function isVisible(string $status): bool
    {
        return in_array($status, ['active', 'published'], true);
    }
}
```

Похожий код не всегда означает одинаковое знание. `active` для пользователя и `published` для товара могут меняться независимо.

Senior-level ответ: DRY нужно балансировать с AHA/WET. Иногда лучше временно оставить повторение, пока не станет понятна настоящая абстракция. Преждевременное объединение разных бизнес-концепций создает более дорогую связанность.

## YAGNI - You Aren't Gonna Need It

Не реализуй функциональность, которая не нужна сейчас. Особенно: расширяемость, конфигурацию, абстракции и generic-механизмы "на будущее".

Пример:

```php
// Не нужно, если сейчас есть только Stripe и нет планов подключать другие провайдеры.
interface PaymentProviderPluginRegistryFactoryResolver
{
    public function resolve(string $tenant): PaymentProviderPluginRegistry;
}
```

Лучше начать с простого сервиса и выделить интерфейс, когда появится реальная вариативность или граница:

```php
final class StripePaymentGateway
{
    public function charge(Invoice $invoice): PaymentResult
    {
        // Stripe API call
    }
}
```

Senior-level ответ: YAGNI снижает стоимость поддержки. Но он не отменяет архитектурные решения, которые сложно добавить позже: границы модулей, миграции данных, безопасность, observability, совместимость публичного API.

## GRASP basics

GRASP - General Responsibility Assignment Software Patterns, набор принципов распределения ответственностей из книги Craig Larman.

Ключевые идеи:

- Information Expert: ответственность получает объект, у которого есть нужные данные.
- Creator: объект создает другой объект, если содержит, агрегирует или тесно использует его.
- Controller: системные события принимает use case/controller, но не делает всю бизнес-логику сам.
- Low Coupling: минимизировать зависимость между модулями.
- High Cohesion: держать связанные обязанности вместе.
- Polymorphism: вариативное поведение лучше отдавать полиморфным типам, а не `switch`/`match` по типу.
- Pure Fabrication: искусственный сервис допустим, если он снижает связанность и повышает связность.
- Indirection: посредник помогает разорвать прямую зависимость.
- Protected Variations: защищать стабильный код от изменчивых точек через интерфейсы, адаптеры, стратегии.

PHP-пример Information Expert:

```php
final class Order
{
    /** @var list<OrderLine> */
    private array $lines = [];

    public function total(): Money
    {
        return array_reduce(
            $this->lines,
            fn (Money $total, OrderLine $line): Money => $total->add($line->subtotal()),
            Money::zero(),
        );
    }
}
```

Не стоит выносить расчет total в `OrderHelper`, если `Order` сам владеет строками заказа.

## Law of Demeter

Объект должен общаться только с ближайшими друзьями: своими полями, параметрами, созданными объектами и прямыми зависимостями. Не стоит строить длинные цепочки доступа к внутренностям.

Антипример:

```php
$country = $order->customer()->profile()->address()->country()->isoCode();
```

Лучше:

```php
$country = $order->shippingCountryCode();
```

Senior-level ответ: Law of Demeter уменьшает знание о внутренней структуре объектов. Но в fluent interfaces, query builders и immutable value objects цепочки могут быть нормальными, если это осознанный API, а не вытекание внутренностей домена.

## Composition over inheritance

Композиция предпочтительнее наследования, когда нужно переиспользовать поведение или комбинировать возможности. Наследование лучше оставлять для устойчивого отношения "is-a" и корректного LSP.

Антипример:

```php
class CsvReport extends FileReport
{
    // Наследование только ради методов saveToDisk() и archive().
}
```

Лучше:

```php
final class CsvReport
{
    public function __construct(private ReportStorage $storage) {}

    public function export(ReportData $data): void
    {
        $this->storage->save('report.csv', $this->render($data));
    }
}
```

В PHP композиция часто удобнее через DI, traits - осторожно. Trait переиспользует код, но не задает контракт и может скрывать зависимости.

## High cohesion / low coupling

High cohesion: внутри модуля находятся связанные обязанности. Low coupling: модуль мало знает о других модулях.

Признаки низкой cohesion:

- класс называется `CommonService`, `Utils`, `Manager`;
- методы работают с разными наборами зависимостей;
- изменение одной бизнес-фичи затрагивает большой универсальный класс.

Признаки высокой coupling:

- домен зависит от Laravel Request, Eloquent, Redis, SMTP;
- тест требует поднять БД для проверки простой бизнес-логики;
- изменение DTO ломает много слоев.

Senior-level ответ: cohesion и coupling - базовые метрики качества дизайна. SOLID, GRASP и SoC в итоге работают на повышение cohesion и снижение harmful coupling.

## Tell Don't Ask

Не вытаскивай данные из объекта, чтобы принять решение снаружи. Скажи объекту, что нужно сделать, и пусть он сам применит свои инварианты.

Антипример:

```php
if ($account->balance()->greaterThanOrEqual($amount)) {
    $account->setBalance($account->balance()->subtract($amount));
}
```

Лучше:

```php
$account->withdraw($amount);
```

```php
final class Account
{
    public function withdraw(Money $amount): void
    {
        if ($this->balance->lessThan($amount)) {
            throw new InsufficientFunds();
        }

        $this->balance = $this->balance->subtract($amount);
    }
}
```

Tradeoff: для read models, DTO, API resources и отчетов ask-подход нормален. Tell Don't Ask важнее в доменной модели с инвариантами.

## Fail Fast

Ошибки нужно обнаруживать как можно раньше и ближе к источнику. Не маскировать некорректное состояние.

Антипример:

```php
final class Email
{
    public function __construct(private string $value) {}
}
```

Лучше:

```php
final readonly class Email
{
    private function __construct(private string $value) {}

    public static function fromString(string $value): self
    {
        if (filter_var($value, FILTER_VALIDATE_EMAIL) === false) {
            throw new InvalidArgumentException('Invalid email');
        }

        return new self($value);
    }

    public function value(): string
    {
        return $this->value;
    }
}
```

Senior-level ответ: Fail Fast улучшает диагностику и защищает инварианты. Но для пользовательского ввода нужны нормальные validation errors, а не 500. Внутри домена исключение уместно, на границе приложения его нужно преобразовать в понятный ответ.

## Separation of Concerns

Разделяй разные concerns: доменная логика, application use cases, инфраструктура, presentation, persistence, integration.

Пример плохого смешения:

```php
final class CheckoutController
{
    public function __invoke(Request $request): JsonResponse
    {
        $order = Order::findOrFail($request->integer('order_id'));
        $order->status = 'paid';
        $order->save();

        Http::post('https://crm.example/orders', $order->toArray());

        return new JsonResponse(['ok' => true]);
    }
}
```

Лучше:

```php
final class CheckoutController
{
    public function __construct(private CheckoutHandler $handler) {}

    public function __invoke(CheckoutRequest $request): JsonResponse
    {
        $this->handler->handle(new CheckoutCommand($request->orderId()));

        return new JsonResponse(['ok' => true]);
    }
}
```

Контроллер разбирает HTTP, use case управляет сценарием, домен проверяет инварианты, инфраструктура делает внешние вызовы.

## Частые компромиссы

### DRY vs premature abstraction

Если два куска кода похожи, но причины изменения разные, объединение создаст coupling. Дублирование можно терпеть, пока не ясна общая модель. Хорошая абстракция появляется после 2-3 реальных случаев, а не после первого совпадения строк.

### OCP vs overengineering

OCP полезен в местах регулярной вариативности. Если изменение редкое и локальное, простая правка `match` может быть дешевле и понятнее. Абстракция должна окупаться снижением риска, а не красотой диаграммы.

### DIP vs interface for everything

Интерфейс с одной реализацией может быть полезен на границе с инфраструктурой или внешним сервисом. Но интерфейс для каждого класса без причины засоряет код и усложняет навигацию.

### SRP vs excessive fragmentation

Один понятный use case class лучше, чем цепочка из мелких классов, где каждый делает одну строку. Делить стоит по причинам изменения, а не по количеству строк.

### KISS vs strategic design

KISS не запрещает DDD, CQRS, hexagonal architecture или event-driven подход. Он требует, чтобы сложность была необходимой и объяснимой бизнесом, нагрузкой, командой или требованиями к изменяемости.

## Мини-практика: рефакторинг

### 1. Убрать нарушение SRP и SoC

До:

```php
final class ReportController
{
    public function download(Request $request): Response
    {
        $rows = DB::table('orders')->where('status', 'paid')->get();
        $csv = implode("\n", $rows->map(fn ($row) => $row->id . ',' . $row->total)->all());
        Storage::put('reports/orders.csv', $csv);

        return response()->download(storage_path('reports/orders.csv'));
    }
}
```

Что сделать:

- `ReportController` оставить HTTP-слоем;
- вынести сценарий в `GeneratePaidOrdersReportHandler`;
- выборку спрятать за `PaidOrdersReportQuery`;
- форматирование CSV вынести в `CsvReportRenderer`.

Короткий ответ на интервью: контроллер не должен знать SQL, формат CSV и файловое хранилище одновременно. Это разные причины изменения.

### 2. Найти ложный DRY

До:

```php
final class StatusHelper
{
    public static function isFinal(string $status): bool
    {
        return in_array($status, ['done', 'cancelled'], true);
    }
}
```

Используется для `Order`, `Task`, `Payment`.

Проблема: одинаковые строки статусов не доказывают одинаковый lifecycle. Лучше отдельные enum/value object или методы доменных объектов.

### 3. Заменить наследование композицией

До:

```php
abstract class AbstractApiClient
{
    protected function request(string $method, string $url): array
    {
        // HTTP details
    }
}

final class CrmClient extends AbstractApiClient {}
final class BillingClient extends AbstractApiClient {}
```

После:

```php
final class CrmClient
{
    public function __construct(private HttpClientInterface $http) {}
}

final class BillingClient
{
    public function __construct(private HttpClientInterface $http) {}
}
```

Так клиенты не связаны общей иерархией и могут независимо менять поведение.

## Вопросы для самопроверки

1. Что означает "одна причина для изменения" в SRP?
2. Когда OCP оправдан, а когда это overengineering?
3. Приведи пример нарушения LSP без классического Rectangle/Square.
4. Чем ISP отличается от простого дробления интерфейсов?
5. Почему DIP не означает "интерфейс для каждого класса"?
6. Чем дублирование кода отличается от дублирования знания?
7. Когда нарушение DRY лучше, чем преждевременная абстракция?
8. Как Law of Demeter связан с encapsulation?
9. Почему композиция часто безопаснее наследования?
10. Где Tell Don't Ask неуместен?
11. Как Fail Fast сочетается с validation errors для пользователя?
12. Как объяснить high cohesion и low coupling на примере Laravel/Symfony приложения?

## Короткие senior-level ответы

**SRP:** класс должен меняться по одной причине. Я смотрю не на количество методов, а на оси изменений: бизнес-правила, persistence, transport, formatting, integration.

**OCP:** стабильный код не должен переписываться при каждом новом варианте поведения. Но я ввожу extension point только там, где вариативность реальна или цена изменения высока.

**LSP:** наследник должен сохранять контракт базового типа. Если подтип кидает `NotSupportedException` или требует особого порядка вызовов, это сигнал нарушения.

**ISP:** клиент должен зависеть только от нужной части контракта. Это снижает coupling и упрощает тесты.

**DIP:** бизнес/use case зависит от абстракции, а инфраструктура является деталью. В PHP это обычно constructor injection плюс интерфейсы на границах.

**KISS:** простое решение - то, которое команда быстро понимает и безопасно меняет. Паттерны не должны добавлять случайную сложность.

**DRY:** устранять нужно дублирование знания, а не каждое совпадение строк. Иначе можно связать разные бизнес-концепции.

**YAGNI:** не строю расширяемость без текущей потребности, но заранее учитываю решения, которые дорого менять позже: данные, API, безопасность, границы модулей.

**GRASP:** это набор эвристик для назначения ответственностей: Information Expert, Creator, Controller, Low Coupling, High Cohesion и другие.

**Law of Demeter:** не нужно заставлять клиент знать внутреннюю структуру объекта. Лучше выразительный метод на ближайшем объекте.

**Composition over inheritance:** композиция гибче и реже ломает LSP. Наследование использую только при настоящем отношении "is-a".

**Fail Fast:** некорректное состояние должно обнаруживаться рано. На границе приложения ошибки превращаются в понятный ответ, внутри домена защищают инварианты.

## Чеклист для code review

- У класса понятная ответственность и причина изменения.
- Бизнес-правила не размазаны между контроллером, моделью, listener и job.
- Нет `ManagerService`, который знает обо всем.
- Новая абстракция имеет понятную причину: вариативность, граница, тестируемость, снижение coupling.
- Интерфейс не создан автоматически без потребности.
- Наследование не используется только ради переиспользования кода.
- Подтип не нарушает контракт родителя и не кидает неожиданный `NotSupportedException`.
- Повторение кода проверено: это дублирование знания или случайное сходство?
- Нет преждевременной универсализации под гипотетическое будущее.
- Доменная логика не зависит напрямую от HTTP, ORM, framework facade, Redis, SMTP.
- Внешние интеграции спрятаны за адаптерами или gateway-классами.
- Объекты защищают свои инварианты, а не отдают все данные наружу.
- Ошибки валидируются рано и преобразуются в корректный response на границе.
- Длинные цепочки вызовов не раскрывают внутреннюю структуру домена.
- Простота решения соответствует сложности задачи.

## Ссылки

- Robert C. Martin, "Design Principles and Design Patterns" - оригинальное изложение принципов SOLID: https://web.archive.org/web/20150906155800/http://www.objectmentor.com/resources/articles/Principles_and_Patterns.pdf
- Robert C. Martin, "The Principles of OOD" - материалы Object Mentor: https://web.archive.org/web/20150906155800/http://butunclebob.com/ArticleS.UncleBob.PrinciplesOfOod
- Barbara Liskov, Jeannette Wing, "A Behavioral Notion of Subtyping" - основа LSP: https://www.cs.cmu.edu/~wing/publications/LiskovWing94.pdf
- Martin Fowler, "Inversion of Control Containers and the Dependency Injection pattern": https://martinfowler.com/articles/injection.html
- Martin Fowler, "TellDontAsk": https://martinfowler.com/bliki/TellDontAsk.html
- Martin Fowler, "Yagni": https://martinfowler.com/bliki/Yagni.html
- Martin Fowler, "CodeSmell": https://martinfowler.com/bliki/CodeSmell.html
- The Pragmatic Programmer, Andrew Hunt, David Thomas - источник формулировки DRY: https://pragprog.com/titles/tpp20/the-pragmatic-programmer-20th-anniversary-edition/
- Craig Larman, "Applying UML and Patterns" - GRASP patterns: https://www.informit.com/store/applying-uml-and-patterns-an-introduction-to-object-oriented-9780131489066
- GRASP overview, University of Washington course notes: https://courses.cs.washington.edu/courses/cse403/11sp/lectures/lecture15-grasp.pdf
- Law of Demeter, Northeastern University: https://www2.ccs.neu.edu/research/demeter/demeter-method/LawOfDemeter/general-formulation.html
- Eric Evans, "Domain-Driven Design" - разделение домена и инфраструктуры: https://www.domainlanguage.com/ddd/
- PHP-FIG, PSR-11 Container Interface - полезно для обсуждения DI-контейнеров в PHP: https://www.php-fig.org/psr/psr-11/
- Symfony Service Container documentation: https://symfony.com/doc/current/service_container.html
- Laravel Service Container documentation: https://laravel.com/docs/container
