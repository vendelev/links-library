# Design Principles: SOLID, KISS, DRY and Responsibility Design

<!-- markdownlint-disable MD013 -->

Практический конспект для senior/lead PHP backend interviews. Фокус: объяснить принципы без догматизма, показать пользу, компромиссы и типичные злоупотребления.

## Как отвечать на интервью

Хороший ответ по принципу проектирования содержит:

- краткое определение;
- проблему, которую принцип решает;
- пример из реального backend-кода;
- trade-off или типичное злоупотребление;
- связь с тестируемостью, изменяемостью и стоимостью сопровождения.

Плохой ответ - просто расшифровать аббревиатуру. Хороший ответ - объяснить, как принцип снижает риск изменения бизнес-логики, инфраструктуры или интеграций.

## SOLID

SOLID - набор эвристик объектно-ориентированного проектирования. Это не чеклист синтаксиса, а способ управлять зависимостями и изменениями.

### SRP - Single Responsibility Principle

Класс должен иметь одну причину для изменения. Ответственность - это не один метод, а одна ось изменений.

Плохой пример:

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
}
```

Проблема: создание счета, генерация PDF, storage и email меняются по разным причинам.

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

Senior-level мысль: SRP помогает локализовать изменения, но дробить каждый метод в отдельный класс не нужно. Слишком мелкая декомпозиция ухудшает навигацию и повышает когнитивную нагрузку.

Типичные ошибки:

- путать SRP с правилом "класс должен иметь один метод";
- создавать `Manager`, `Helper`, `Service`, которые делают все;
- преждевременно разносить простой use case на десятки классов.

### OCP - Open/Closed Principle

Модуль должен быть открыт для расширения и закрыт для изменения. Новое поведение добавляется без переписывания стабильного кода.

Плохой пример:

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

Вариант через Strategy:

```php
interface DiscountPolicy
{
    public function supports(Order $order): bool;
    public function discountFor(Order $order): Money;
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

Senior-level мысль: OCP оправдан, когда есть стабильная точка расширения и реальная вариативность: платежные провайдеры, способы доставки, правила ценообразования. Для одноразовой ветки `if` Strategy может быть overengineering.

### LSP - Liskov Substitution Principle

Подтип должен быть взаимозаменяем с базовым типом без нарушения корректности программы. Наследник не должен усиливать предусловия, ослаблять постусловия или неожиданно менять семантику.

Плохой пример:

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
```

LSP проявляется в контрактах: исключения, nullable-результаты, побочные эффекты, единицы измерения, порядок вызовов, транзакционные гарантии.

Типичные ошибки:

- наследник кидает `NotSupportedException` для метода родителя;
- наследник требует более строгий формат входных данных;
- наследник возвращает `null`, хотя базовый контракт обещает объект;
- наследование используется ради переиспользования кода, а не ради отношения `is-a`.

### ISP - Interface Segregation Principle

Клиенты не должны зависеть от методов, которые они не используют. Лучше несколько узких интерфейсов, чем один жирный.

Плохой пример:

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
```

Senior-level мысль: ISP уменьшает лишние зависимости и упрощает test doubles. Но интерфейс для каждого класса автоматически создавать не нужно. Интерфейс полезен, когда есть несколько реализаций, граница слоя, контракт для внешнего клиента или нужно изолировать инфраструктуру.

### DIP - Dependency Inversion Principle

Высокоуровневые модули не должны зависеть от низкоуровневых деталей. Оба должны зависеть от абстракций. Абстракции не должны зависеть от деталей.

Плохой пример:

```php
final class RegisterUserHandler
{
    public function handle(RegisterUserCommand $command): void
    {
        $user = new User($command->email);
        $pdo = new PDO($_ENV['DSN']);
        $pdo->prepare('INSERT INTO users ...')->execute([]);
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

DIP - не про интерфейсы везде, а про направление зависимостей. Доменная логика не должна знать про PDO, HTTP client, Redis, SMTP или Laravel Facade, если это затрудняет тестирование и замену деталей.

В PHP DIP часто реализуется через:

- constructor injection;
- Symfony/Laravel service container;
- интерфейсы на границах приложения;
- adapters для внешних сервисов;
- ports and adapters в Hexagonal Architecture.

## KISS

KISS - выбирать самое простое решение, которое корректно закрывает текущую задачу и понятно команде.

Антипример:

```php
interface ActiveUserSpecificationFactoryInterface
{
    public function create(): SpecificationInterface;
}
```

Если задача - проверить активность пользователя, достаточно:

```php
if (!$user->isActive()) {
    throw new UserIsNotActive();
}
```

KISS не означает писать примитивно. Простота - это низкая случайная сложность: меньше ненужных слоев, меньше магии, понятные зависимости, явные инварианты.

Признаки нарушения KISS:

- паттерн добавлен только потому, что "так правильно";
- flow невозможно понять без IDE и дебаггера;
- абстракция имеет одну реализацию и не защищает важную границу;
- простая бизнес-операция проходит через много классов без причины.

## DRY

DRY - каждое знание должно иметь единственное, однозначное представление в системе. Это не про удаление всех похожих строк, а про устранение дублирования знания.

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

Senior-level мысль: DRY нужно балансировать с AHA/WET. Иногда лучше временно оставить повторение, пока не станет понятна настоящая абстракция. Преждевременное объединение разных бизнес-концепций создает дорогую связанность.

## YAGNI

YAGNI - не реализовывать функциональность, которая не нужна сейчас. Особенно это касается расширяемости, конфигурации, абстракций и generic-механизмов "на будущее".

Пример:

```php
// Не нужно, если сейчас есть только Stripe и нет реальных планов подключать другие провайдеры.
interface PaymentProviderPluginRegistryFactoryResolver
{
    public function resolve(string $tenant): PaymentProviderPluginRegistry;
}
```

YAGNI снижает стоимость поддержки, но не отменяет решения, которые дорого добавить позже: границы модулей, миграции данных, безопасность, observability, совместимость публичного API.

## GRASP

GRASP - General Responsibility Assignment Software Patterns. Это принципы распределения ответственностей.

Ключевые идеи:

- Information Expert: ответственность получает объект, у которого есть нужные данные.
- Creator: объект создает другой объект, если содержит, агрегирует или тесно использует его.
- Controller: системные события принимает use case/controller, но не делает всю бизнес-логику сам.
- Low Coupling: минимизировать зависимость между модулями.
- High Cohesion: держать связанные обязанности вместе.
- Polymorphism: вариативное поведение лучше отдавать полиморфным типам, а не `switch`/`match` по типу.
- Pure Fabrication: искусственный сервис допустим, если он снижает связанность и повышает связность.
- Indirection: посредник помогает разорвать прямую зависимость.
- Protected Variations: защищать стабильный код от изменчивых точек через интерфейсы, adapters, strategies.

Пример Information Expert:

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

Объект должен общаться только с ближайшими друзьями: своими полями, параметрами, созданными объектами и прямыми зависимостями.

Плохой пример:

```php
$country = $order->customer()->profile()->address()->country()->isoCode();
```

Лучше:

```php
$country = $order->shippingCountryCode();
```

Law of Demeter уменьшает знание о внутренней структуре объектов. Но в fluent interfaces, query builders и immutable value objects цепочки могут быть нормальным API.

## Composition Over Inheritance

Композиция предпочтительнее наследования, когда нужно переиспользовать поведение или комбинировать возможности. Наследование лучше оставлять для устойчивого отношения `is-a` и корректного LSP.

Плохой пример:

```php
class CsvReport extends FileReport
{
    // Наследование только ради saveToDisk() и archive().
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

В PHP композиция часто удобнее через DI. Traits переиспользуют код, но не задают контракт и могут скрывать зависимости.

## High Cohesion and Low Coupling

High cohesion: внутри модуля находятся связанные обязанности. Low coupling: модуль мало знает о других модулях.

Признаки низкой cohesion:

- класс называется `CommonService`, `Utils`, `Manager`;
- методы работают с разными наборами зависимостей;
- изменение одной бизнес-фичи затрагивает большой универсальный класс.

Признаки высокой coupling:

- домен зависит от Laravel Request, Eloquent, Redis, SMTP;
- тест требует поднять БД для простой бизнес-логики;
- изменение DTO ломает много слоев.

SOLID, GRASP и SoC в итоге работают на повышение cohesion и снижение harmful coupling.

## Tell, Don't Ask

Не вытаскивать данные из объекта, чтобы принять решение снаружи. Сказать объекту, что нужно сделать, и пусть он сам применит инварианты.

Плохой пример:

```php
if ($account->balance()->greaterThanOrEqual($amount)) {
    $account->setBalance($account->balance()->subtract($amount));
}
```

Лучше:

```php
$account->withdraw($amount);
```

Для read models, DTO, API resources и отчетов ask-подход нормален. Tell Don't Ask важнее в доменной модели с инвариантами.

## Fail Fast

Ошибки нужно обнаруживать как можно раньше и ближе к источнику. Не маскировать некорректное состояние.

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
}
```

Fail Fast улучшает диагностику и защищает инварианты. Для пользовательского ввода нужны validation errors, а не 500. Внутри домена исключение уместно, на границе приложения его нужно преобразовать в понятный response.

## Separation of Concerns

Разделять разные concerns: domain logic, application use cases, infrastructure, presentation, persistence, integration.

Плохой пример:

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

Лучше: controller разбирает HTTP, use case управляет сценарием, домен проверяет инварианты, infrastructure делает внешние вызовы.

## Частые компромиссы

### DRY vs Premature Abstraction

Если два куска кода похожи, но причины изменения разные, объединение создаст coupling. Хорошая абстракция появляется после нескольких реальных случаев, а не после первого совпадения строк.

### OCP vs Overengineering

OCP полезен в местах регулярной вариативности. Если изменение редкое и локальное, простая правка `match` может быть дешевле и понятнее.

### DIP vs Interface for Everything

Интерфейс с одной реализацией может быть полезен на границе с инфраструктурой или внешним сервисом. Но интерфейс для каждого класса без причины усложняет навигацию.

### SRP vs Excessive Fragmentation

Один понятный use case class лучше цепочки мелких классов, где каждый делает одну строку.

### KISS vs Strategic Design

KISS не запрещает DDD, CQRS, Hexagonal Architecture или Event-Driven подход. Он требует, чтобы сложность была необходимой и объяснимой бизнесом, нагрузкой, командой или требованиями к изменяемости.

## Практика рефакторинга

### Убрать нарушение SRP и SoC

Плохой код:

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

### Найти ложный DRY

Если `StatusHelper::isFinal()` используется для `Order`, `Task`, `Payment`, одинаковые строки статусов не доказывают одинаковый lifecycle. Лучше отдельные enum/value object или методы доменных объектов.

### Заменить наследование композицией

Вместо `CrmClient extends AbstractApiClient` и `BillingClient extends AbstractApiClient` лучше передавать общий `HttpClientInterface` через конструктор. Клиенты не связаны общей иерархией и могут независимо менять поведение.

## Code Review Checklist

- У класса понятная ответственность и причина изменения.
- Бизнес-правила не размазаны между controller, model, listener и job.
- Нет `ManagerService`, который знает обо всем.
- Новая абстракция имеет причину: вариативность, граница, тестируемость, снижение coupling.
- Интерфейс не создан автоматически без потребности.
- Наследование не используется только ради переиспользования кода.
- Подтип не нарушает контракт родителя.
- Повторение кода проверено: это дублирование знания или случайное сходство?
- Нет преждевременной универсализации под гипотетическое будущее.
- Доменная логика не зависит напрямую от HTTP, ORM, framework facade, Redis, SMTP.
- Внешние интеграции спрятаны за adapters или gateway-классами.
- Объекты защищают свои инварианты.
- Ошибки валидируются рано и преобразуются в корректный response на границе.
- Длинные цепочки вызовов не раскрывают внутреннюю структуру домена.
- Простота решения соответствует сложности задачи.

## Self-Check Questions

1. Что означает одна причина для изменения в SRP?
2. Когда OCP оправдан, а когда это overengineering?
3. Приведи пример нарушения LSP без Rectangle/Square.
4. Чем ISP отличается от простого дробления интерфейсов?
5. Почему DIP не означает интерфейс для каждого класса?
6. Чем дублирование кода отличается от дублирования знания?
7. Когда нарушение DRY лучше преждевременной абстракции?
8. Как Law of Demeter связан с encapsulation?
9. Почему композиция часто безопаснее наследования?
10. Где Tell Don't Ask неуместен?
11. Как Fail Fast сочетается с validation errors для пользователя?
12. Как объяснить high cohesion и low coupling на примере Laravel/Symfony приложения?

## Дополнительное чтение

- Robert C. Martin, Design Principles and Design Patterns: <https://web.archive.org/web/20150906155800/http://www.objectmentor.com/resources/articles/Principles_and_Patterns.pdf>
- Robert C. Martin, The Principles of OOD: <https://web.archive.org/web/20150906155800/http://butunclebob.com/ArticleS.UncleBob.PrinciplesOfOod>
- Barbara Liskov, Jeannette Wing, A Behavioral Notion of Subtyping: <https://www.cs.cmu.edu/~wing/publications/LiskovWing94.pdf>
- Martin Fowler, Inversion of Control Containers and the Dependency Injection pattern: <https://martinfowler.com/articles/injection.html>
- Martin Fowler, TellDontAsk: <https://martinfowler.com/bliki/TellDontAsk.html>
- Martin Fowler, Yagni: <https://martinfowler.com/bliki/Yagni.html>
- Martin Fowler, CodeSmell: <https://martinfowler.com/bliki/CodeSmell.html>
- The Pragmatic Programmer: <https://pragprog.com/titles/tpp20/the-pragmatic-programmer-20th-anniversary-edition/>
- Craig Larman, Applying UML and Patterns: <https://www.informit.com/store/applying-uml-and-patterns-an-introduction-to-object-oriented-9780131489066>
- GRASP overview, University of Washington course notes: <https://courses.cs.washington.edu/courses/cse403/11sp/lectures/lecture15-grasp.pdf>
- Law of Demeter, Northeastern University: <https://www2.ccs.neu.edu/research/demeter/demeter-method/LawOfDemeter/general-formulation.html>
- PHP-FIG, PSR-11 Container Interface: <https://www.php-fig.org/psr/psr-11/>
- Symfony Service Container: <https://symfony.com/doc/current/service_container.html>
- Laravel Service Container: <https://laravel.com/docs/container>
