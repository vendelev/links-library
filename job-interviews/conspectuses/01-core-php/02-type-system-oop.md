# PHP type system, OOP, exceptions, attributes, enums, readonly, property hooks

Цель: быстро освежить темы, которые часто проверяют у Senior/Lead PHP-разработчика. Фокус на объяснениях для интервью, практических ловушках и коротких примерах на PHP 8.x, включая PHP 8.4.

## Документация и статьи

Официальная документация PHP:

- [PHP Manual: Type declarations](https://www.php.net/manual/en/language.types.declarations.php)
- [PHP Manual: Type system](https://www.php.net/manual/en/language.types.type-system.php)
- [PHP Manual: `strict_types`](https://www.php.net/manual/en/language.types.declarations.php#language.types.declarations.strict)
- [PHP Manual: OOP](https://www.php.net/manual/en/language.oop5.php)
- [PHP Manual: Visibility](https://www.php.net/manual/en/language.oop5.visibility.php)
- [PHP Manual: Object interfaces](https://www.php.net/manual/en/language.oop5.interfaces.php)
- [PHP Manual: Traits](https://www.php.net/manual/en/language.oop5.traits.php)
- [PHP Manual: Exceptions](https://www.php.net/manual/en/language.exceptions.php)
- [PHP Manual: Errors in PHP 7+](https://www.php.net/manual/en/language.errors.php7.php)
- [PHP Manual: Attributes](https://www.php.net/manual/en/language.attributes.php)
- [PHP Manual: Enumerations](https://www.php.net/manual/en/language.enumerations.php)
- [PHP Manual: Readonly properties](https://www.php.net/manual/en/language.oop5.properties.php#language.oop5.properties.readonly-properties)
- [PHP Manual: Property hooks](https://www.php.net/manual/en/language.oop5.property-hooks.php)
- [PHP Manual: Asymmetric property visibility](https://www.php.net/manual/en/language.oop5.visibility.php#language.oop5.visibility-members-asymmetric)

PHP RFCs:

- [RFC: Scalar Type Declarations](https://wiki.php.net/rfc/scalar_type_hints_v5)
- [RFC: Return Type Declarations](https://wiki.php.net/rfc/return_types)
- [RFC: Nullable Types](https://wiki.php.net/rfc/nullable_types)
- [RFC: Union Types 2.0](https://wiki.php.net/rfc/union_types_v2)
- [RFC: Pure Intersection Types](https://wiki.php.net/rfc/pure-intersection-types)
- [RFC: DNF Types](https://wiki.php.net/rfc/dnf_types)
- [RFC: Covariant Returns and Contravariant Parameters](https://wiki.php.net/rfc/covariant-returns-and-contravariant-parameters)
- [RFC: Attributes v2](https://wiki.php.net/rfc/attributes_v2)
- [RFC: Enumerations](https://wiki.php.net/rfc/enumerations)
- [RFC: Readonly Properties 2.0](https://wiki.php.net/rfc/readonly_properties_v2)
- [RFC: Readonly Classes](https://wiki.php.net/rfc/readonly_classes)
- [RFC: Property Hooks](https://wiki.php.net/rfc/property-hooks)
- [RFC: Asymmetric Visibility](https://wiki.php.net/rfc/asymmetric-visibility-v2)

Статический анализ и generics:

- [PHPStan: PHPDoc Types](https://phpstan.org/writing-php-code/phpdoc-types)
- [PHPStan: Generics in PHP using PHPDocs](https://phpstan.org/blog/generics-in-php-using-phpdocs)
- [PHPStan: Generics by Examples](https://phpstan.org/blog/generics-by-examples)
- [Psalm: Supported Annotations](https://psalm.dev/docs/annotating_code/supported_annotations/)
- [Psalm: Templated Annotations](https://psalm.dev/docs/annotating_code/templated_annotations/)
- [Psalm: Type Syntax](https://psalm.dev/docs/annotating_code/type_syntax/)

Практические статьи:

- [Stitcher.io: PHP 8 attributes](https://stitcher.io/blog/attributes-in-php-8)
- [Stitcher.io: PHP 8.1 enums](https://stitcher.io/blog/php-enums)
- [Stitcher.io: PHP 8.1 readonly properties](https://stitcher.io/blog/php-81-readonly-properties)
- [Symfony Docs: Attributes overview](https://symfony.com/doc/current/reference/attributes.html)
- [Laravel Docs: Eloquent enum casting](https://laravel.com/docs/eloquent-mutators#enum-casting)

## PHP Type System

PHP движется от динамического языка к гибридной модели: runtime-типы в сигнатурах плюс статический анализ через PHPStan/Psalm. Типы в PHP - это не только документация, но и контракт API, LSP, refactoring safety и снижение runtime-дефектов.

Базовые группы типов:

- Скалярные: `int`, `float`, `string`, `bool`.
- Специальные: `null`, `false`, `true`, `void`, `never`, `mixed`.
- Объектные: class/interface names, `object`, `self`, `static`, `parent`.
- Составные: nullable, union, intersection, DNF.

```php
declare(strict_types=1);

interface Logger
{
    public function info(string $message): void;
}

final class UserService
{
    public function __construct(private Logger $logger) {}

    public function findUserId(string $email): int|false
    {
        if ($email === '') {
            return false;
        }

        return 42;
    }
}
```

Senior-ответ: типы в PHP проверяются во время выполнения, но не покрывают все инварианты предметной области. `string $email` не гарантирует валидный email, поэтому нужны value objects, валидаторы или статический анализ.

### `strict_types`

`declare(strict_types=1)` влияет на coercion скалярных аргументов в вызовах функций из текущего файла. Он не делает PHP полностью статически типизированным.

Важно: `strict_types` определяется в файле вызывающего кода, а не в файле, где функция объявлена.

```php
// library.php
declare(strict_types=1);

function takesInt(int $value): void {}

// caller.php без strict_types
require 'library.php';

takesInt('123'); // будет приведено к int, потому что caller.php не strict
```

Ловушки:

- `strict_types` не проверяет содержимое массивов.
- `strict_types` не превращает PHP в compile-time checked language.
- Internal functions исторически имели собственные правила coercion; современный PHP стал строже, но legacy-нюансы встречаются.

### Nullable, Union, Intersection, DNF

`?T` равен `T|null`, но нельзя писать `?A|B`; нужно `A|B|null`.

Union тип означает: значение должно соответствовать хотя бы одному типу.

```php
function normalizeId(int|string $id): string
{
    return (string) $id;
}
```

Intersection тип означает: значение должно соответствовать всем указанным типам.

```php
function store(Cacheable&JsonSerializable $resource): void
{
    $key = $resource->cacheKey();
    $json = json_encode($resource, JSON_THROW_ON_ERROR);
}
```

DNF, disjunctive normal form, позволяет комбинировать union и intersection: `(A&B)|C`.

```php
function schedule((HasId&CanBeExported)|ExportJob $item): void
{
    // Принимаем либо объект с двумя контрактами, либо готовую job.
}
```

Ловушки:

- `int|float` может усложнять арифметику и сериализацию.
- `array|Traversable` часто лучше заменить на `iterable`.
- `false` в union полезен для legacy API, но в новом коде часто лучше exception или `Result`.
- `mixed` в union бессмысленен, потому что уже включает все.
- DNF может ухудшать читаемость; если тип трудно объяснить, возможно нужен отдельный интерфейс или value object.

### `void`, `never`, `mixed`, `object`

`void` означает отсутствие полезного возвращаемого значения.

`never` означает, что функция не возвращает управление: бросает exception, вызывает `exit`, бесконечно выполняется.

`mixed` означает любой тип и требует narrowing перед использованием.

`object` означает любой объект, но без гарантий конкретных методов.

```php
function fail(string $message): never
{
    throw new RuntimeException($message);
}

function handle(mixed $payload): void
{
    if (!is_array($payload)) {
        fail('Array expected');
    }

    // Здесь статический анализатор может понимать, что $payload is array.
}
```

## Variance

Variance описывает совместимость типов при наследовании методов.

- Covariance: возвращаемый тип в наследнике можно сужать.
- Contravariance: тип параметра в наследнике можно расширять.
- Invariance: для properties тип должен совпадать, потому что свойство читается и записывается.

```php
class Animal {}
class Dog extends Animal {}

interface Shelter
{
    public function adopt(): Animal;
    public function accept(Dog $dog): void;
}

final class DogShelter implements Shelter
{
    public function adopt(): Dog
    {
        return new Dog();
    }

    public function accept(Animal $dog): void
    {
    }
}
```

Ловушки:

- Нельзя сузить parameter type в наследнике: это нарушит LSP.
- Нельзя расширить return type в наследнике: клиент получит меньше гарантий.
- Properties инвариантны, кроме случаев `readonly`/asymmetric visibility, где меняются возможности записи.

## Generics через PHPDoc

В PHP нет native generics, но PHPStan и Psalm поддерживают generics через аннотации. Это критично для коллекций, репозиториев, DTO mapping и service locators.

```php
/**
 * @template T
 */
final class Collection
{
    /** @var list<T> */
    private array $items;

    /**
     * @param list<T> $items
     */
    public function __construct(array $items)
    {
        $this->items = $items;
    }

    /**
     * @return T|null
     */
    public function first(): mixed
    {
        return $this->items[0] ?? null;
    }
}
```

Практические PHPDoc-типы:

- `list<T>`: массив с последовательными integer keys от 0.
- `array<string, T>`: map.
- `non-empty-array` и `non-empty-list`.
- `class-string<T>`: строка с именем класса.
- `key-of<T>` и `value-of<T>` для shape-like контрактов.
- `array{foo: string, bar?: int}`: array shape.

Senior-ответ: PHPDoc generics не проверяются runtime, но дают сильную проверку на CI. Их надо использовать там, где native type system не выражает контракт: коллекции, factories, repositories, serializers.

## OOP в PHP

Senior/Lead должен объяснять OOP не как набор ключевых слов, а как управление зависимостями, изменяемостью, инвариантами и границами модулей.

### Visibility

- `public`: часть внешнего контракта.
- `protected`: контракт для наследников; часто создает сильную связанность.
- `private`: локальная реализация класса.

Ловушка: `protected` часто выглядит удобно, но ломает encapsulation. Для extension points обычно лучше интерфейс, композиция или template method с узким контрактом.

### Inheritance

Наследование должно моделировать substitutability, а не reuse кода.

```php
abstract class PaymentProvider
{
    final public function charge(Money $money): PaymentResult
    {
        $this->validate($money);

        return $this->doCharge($money);
    }

    protected function validate(Money $money): void {}

    abstract protected function doCharge(Money $money): PaymentResult;
}
```

Senior-ответ: `final` на публичном алгоритме защищает инвариант, а наследникам оставляет только конкретную операцию. Если вариантов поведения много и они меняются независимо, стратегия через композицию чаще лучше.

### Interfaces

Интерфейс описывает роль, а не просто зеркало класса.

```php
interface Transactional
{
    /**
     * @template T
     * @param callable(): T $callback
     * @return T
     */
    public function transaction(callable $callback): mixed;
}
```

Ловушки:

- Interface с одним implementation не всегда плох, если это boundary: DB, queue, HTTP client, clock, filesystem.
- Interface ради тестов без архитектурной границы может быть лишней абстракцией.
- Breaking change в interface дороже, чем в class.

### Traits

Trait - механизм reuse кода, не тип. Он может содержать state, методы и требования к классу, но не должен скрывать сложную зависимость.

Senior-ответ: trait оправдан для горизонтальной технической возможности, например domain events или timestamp helpers. Если trait требует много скрытых методов и properties от класса, лучше явная композиция.

### Composition

Композиция обычно лучше наследования для вариативного поведения.

```php
interface TaxCalculator
{
    public function taxFor(Money $net): Money;
}

final class InvoiceService
{
    public function __construct(private TaxCalculator $taxCalculator) {}

    public function total(Money $net): Money
    {
        return $net->add($this->taxCalculator->taxFor($net));
    }
}
```

Interview explanation: composition делает зависимость явной, облегчает замену стратегии, тестирование и соблюдение SRP.

## Exceptions and Errors Model

Throwable hierarchy:

```text
Throwable
├── Exception
│   ├── RuntimeException
│   ├── LogicException
│   └── ...
└── Error
    ├── TypeError
    ├── ValueError
    ├── ParseError
    └── ...
```

`Exception` обычно для бизнес- и инфраструктурных сбоев.

`Error` обычно для ошибок движка или нарушений типа/значения.

```php
try {
    $payload = json_decode($json, true, flags: JSON_THROW_ON_ERROR);
} catch (JsonException $e) {
    throw new InvalidPayload('Invalid JSON', previous: $e);
}
```

Senior-подход:

- Не ловить `Throwable` на низком уровне без веской причины.
- Конвертировать infrastructure exceptions в domain/application exceptions на границах слоя.
- Сохранять `previous`, чтобы не терять root cause.
- Не использовать exception для обычной ветки control flow в горячем пути.
- Для invariant violation использовать `LogicException`/`InvalidArgumentException` или domain-specific exception.

Ловушки:

- `finally` выполняется и после `return`, но может переопределить exception/return, если сам бросит exception или вернет значение.
- `catch (Exception)` не поймает `TypeError`; нужен `Throwable`, но использовать осторожно.
- `ValueError` часто бросают internal functions при неверном значении корректного типа.

## Attributes

Attributes - native metadata, доступная через Reflection. Они заменяют часть DocBlock-аннотаций, когда metadata нужна runtime.

```php
#[Attribute(Attribute::TARGET_CLASS | Attribute::IS_REPEATABLE)]
final class Route
{
    public function __construct(
        public string $method,
        public string $path,
    ) {}
}

#[Route('GET', '/users')]
#[Route('POST', '/users')]
final class UserController
{
}
```

Senior-ответ: attribute сам по себе ничего не делает. Поведение появляется только в коде, который читает Reflection metadata: framework, compiler pass, custom loader.

Ловушки:

- Attribute constructor выполняется при `newInstance()`, а не при объявлении класса.
- Не стоит переносить в attributes сложную бизнес-логику.
- Attributes хороши для декларативной metadata: routes, validation, serialization groups, DI tags, ORM mapping.
- Для чисто статической информации PHPDoc может быть лучше и дешевле.

## Enums

PHP enums бывают pure и backed.

```php
enum OrderStatus: string
{
    case Draft = 'draft';
    case Paid = 'paid';
    case Cancelled = 'cancelled';

    public function canBeCancelled(): bool
    {
        return match ($this) {
            self::Draft, self::Paid => true,
            self::Cancelled => false,
        };
    }
}
```

`from()` бросает `ValueError`, `tryFrom()` возвращает `null`.

```php
$status = OrderStatus::tryFrom($input)
    ?? throw new InvalidArgumentException('Unknown order status');
```

Senior-ответ: enum хорош для закрытого набора значений с поведением. Если набор расширяется пользователем или хранится в БД как справочник, enum может быть слишком жестким.

Ловушки:

- Enum case - singleton object, сравнивать лучше через `===`.
- Backed enum value должен быть `int` или `string`.
- Enums не могут содержать properties, но могут иметь методы, constants и traits без state.
- При сериализации API надо явно решить: отдавать `name`, `value` или DTO-представление.

## Readonly Properties and Classes

`readonly` property можно записать один раз из области класса, после чего нельзя изменить.

```php
final class UserId
{
    public function __construct(public readonly int $value)
    {
        if ($value <= 0) {
            throw new InvalidArgumentException('User id must be positive');
        }
    }
}
```

`readonly class` делает все instance properties readonly.

```php
readonly class Address
{
    public function __construct(
        public string $city,
        public string $street,
    ) {}
}
```

Важная ловушка: `readonly` не делает вложенные объекты immutable.

```php
final class Profile
{
    public function __construct(public readonly DateTime $createdAt) {}
}

$profile = new Profile(new DateTime());
$profile->createdAt->modify('+1 day'); // объект внутри изменился
```

Senior-ответ: `readonly` защищает от переназначения property, но не гарантирует deep immutability. Для настоящей immutability нужны immutable dependencies, отсутствие mutating methods и осторожная работа с коллекциями.

## Property Hooks and Asymmetric Visibility

Property hooks и asymmetric visibility появились в PHP 8.4. На интервью важно понимать идею даже если проект еще на PHP 8.1-8.3.

### Property Hooks

Property hooks позволяют определить логику `get` и `set` прямо на property, сохраняя синтаксис обращения к свойству.

```php
final class Person
{
    public string $name {
        set {
            if (trim($value) === '') {
                throw new InvalidArgumentException('Name cannot be empty');
            }

            $this->name = trim($value);
        }
    }
}
```

Computed property:

```php
final class Rectangle
{
    public function __construct(
        public int $width,
        public int $height,
    ) {}

    public int $area {
        get => $this->width * $this->height;
    }
}
```

Практический смысл:

- Валидация и нормализация assignment без явных setters.
- Computed read-only properties без boilerplate getters.
- Более явные DTO/value object контракты.

Ловушки:

- Hooks не должны скрывать дорогие I/O операции.
- Assignment внутри `set` должен быть аккуратным, чтобы не получить рекурсию или нарушение правил backed/virtual property.
- Если логика сложная, обычный метод может быть честнее.

### Asymmetric Visibility

Asymmetric visibility позволяет сделать чтение шире, чем запись.

```php
final class Account
{
    public private(set) string $status = 'new';

    public function activate(): void
    {
        $this->status = 'active';
    }
}
```

Senior-ответ: asymmetric visibility уменьшает необходимость в trivial getters и защищает инварианты записи. Это не замена domain methods: если переход состояния имеет смысл, оставляем метод `activate()`, `cancel()`, `pay()`.

Ловушки:

- Public read не означает, что состояние можно менять снаружи.
- API становится похожим на DTO, но запись контролируется классом.
- Для сложных state transitions лучше explicit methods, а не открытый `set` hook.

## Common Interview Pitfalls

1. Путать `strict_types` файла объявления и файла вызова.
2. Использовать `mixed` как способ избежать проектирования контракта.
3. Считать `readonly` полной immutability.
4. Ловить `Exception`, ожидая поймать `TypeError`.
5. Создавать interface для каждого класса без архитектурной границы.
6. Использовать trait как скрытый service locator.
7. Делать enum для открытого справочника, который должен расширяться без deploy.
8. Злоупотреблять union/DNF вместо введения доменного типа.
9. Игнорировать variance при изменении interface.
10. Переносить runtime metadata в PHPDoc или static metadata в attributes без причины.
