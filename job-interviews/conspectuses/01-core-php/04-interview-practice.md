# Core PHP: вопросы, короткие ответы и мини-практика

Цель: проговорить материал как на собеседовании. Этот файл собран из вопросов, коротких ответов и практики исходных конспектов без повторения полной теории.

## Вопросы по PHP versions

1. Чем `match` отличается от `switch`?
2. Почему named arguments могут сломать backward compatibility библиотеки?
3. Что такое union, intersection и DNF types?
4. Чем `mixed` отличается от отсутствия типа?
5. Что реально делает `strict_types=1`?
6. Почему `readonly` не равно deep immutability?
7. Когда использовать enum, а когда достаточно class constants?
8. Что такое Fibers и почему они не равны многопоточности?
9. Почему dynamic properties deprecated - это хорошо для больших проектов?
10. Как мигрировать `function foo(string $x = null)` в PHP 8.4?
11. В чем смысл property hooks и какой у них риск?
12. Где полезна asymmetric visibility?
13. Почему JIT редко ускоряет обычный Laravel/Symfony CRUD?
14. Чем `json_validate()` лучше `json_decode()` для простой проверки валидности?
15. Что проверять в Docker image при обновлении PHP?

## Вопросы по type system и OOP

1. Какой файл определяет поведение `strict_types`: файл функции или файл вызова?
2. Почему `array<User>` нельзя выразить native PHP type?
3. В чем разница между `?Foo` и `Foo|null`?
4. Когда `iterable` лучше `array|Traversable`?
5. Почему нельзя сузить тип параметра в реализации интерфейса?
6. Почему `protected` может ухудшить encapsulation?
7. Что произойдет, если ловить `Exception`, а код бросит `TypeError`?
8. Когда `from()` enum лучше `tryFrom()` и наоборот?
9. Почему attributes не должны содержать бизнес-логику?
10. Как property hooks меняют дизайн DTO и value objects?
11. Чем `public private(set)` отличается от `public readonly`?
12. Когда DNF type лучше заменить отдельным интерфейсом?

## Вопросы по runtime и Composer

1. PHP-FPM создает новый процесс на каждый запрос?
2. Что означает shared-nothing?
3. Почему CLI и FPM могут вести себя по-разному?
4. Как выбрать `pm.max_children`?
5. Зачем `pm.max_requests`?
6. Что делать при `server reached pm.max_children`?
7. Что ускоряет OPcache?
8. Когда нужен `opcache.validate_timestamps=0`?
9. Почему JIT редко ускоряет обычный web CRUD?
10. Что такое copy-on-write?
11. Почему `foreach` по ссылке опасен?
12. Зачем GC, если есть reference counting?
13. Почему long-running workers сложнее FPM?
14. Разница между `composer install` и `composer update`?
15. Нужно ли коммитить `composer.lock`?
16. Что делает `--classmap-authoritative`?
17. Зачем `config.platform.php`?
18. В чем риск Composer scripts/plugins?

## Короткие ответы

### Чем `match` отличается от `switch`?

`match` - expression, возвращает значение, использует строгое сравнение `===`, не имеет fall-through и требует явного покрытия вариантов или `default`. `switch` - statement, использует слабое сравнение `==` и может проваливаться между case без `break`.

### Почему named arguments могут сломать BC?

Потому что имя параметра становится частью публичного контракта. Если пользователь вызывает `foo(timeout: 10)`, то переименование `$timeout` в `$requestTimeout` ломает его код, даже если порядок и тип параметра не изменились.

### Что дает `strict_types=1`?

Он отключает неявное приведение скалярных аргументов для вызовов из текущего файла. Это повышает предсказуемость API, но не делает PHP статически типизированным и не типизирует содержимое массивов.

### Когда использовать union type, а когда value object?

Union подходит для технически простого контракта, например `int|string` input id. Если тип несет доменный смысл, правила валидации или поведение, лучше value object.

### Что такое `readonly` и почему это не deep immutability?

`readonly` запрещает повторно присвоить свойство после инициализации. Но если свойство содержит объект, внутреннее состояние этого объекта может быть изменяемым. Поэтому `readonly` защищает ссылку, но не весь объектный граф.

### Чем `Error` отличается от `Exception`?

Оба реализуют `Throwable`, но `Error` обычно сигнализирует ошибки движка, типов или значений, а `Exception` чаще используется для ожидаемых application/domain/infrastructure failures.

### Зачем нужны PHPDoc generics?

Чтобы выразить типы, которые native PHP пока не умеет: `Collection<User>`, `class-string<T>`, `array<string, DTO>`. Это дает проверку через PHPStan/Psalm и улучшает IDE/refactoring.

### Когда attributes лучше PHPDoc?

Когда metadata нужна runtime и ее читает Reflection-based механизм: routes, DI, validation, ORM mapping. Для статического анализа PHPDoc часто лучше.

### Когда trait уместен?

Когда он добавляет небольшое повторяемое поведение без скрытых зависимостей и сложного состояния. Если trait требует много предположений о классе, лучше композиция.

### Что такое covariance и contravariance?

Covariance позволяет сузить return type в наследнике. Contravariance позволяет расширить parameter type в наследнике. Это нужно для соблюдения LSP.

### Что такое Fibers?

Fibers - механизм cooperative multitasking: выполнение можно приостановить и продолжить позже. Это не потоки и не параллельное выполнение CPU-кода. Fibers полезны как низкоуровневая база для async runtimes, event loop и неблокирующего I/O.

### Как относиться к property hooks?

Это удобный способ выразить computed или validated properties без ручных getter/setter-ов. Но есть риск спрятать тяжелую или неожиданную бизнес-логику за обычным чтением/записью свойства. Для доменной логики по-прежнему лучше явные методы.

### Как исправлять dynamic properties?

Сначала понять назначение объекта. Если это произвольный контейнер данных - возможно, нужен `stdClass` или массив. Если это DTO - объявить свойства явно. Если это ORM/proxy - использовать поддерживаемый механизм фреймворка. `#[AllowDynamicProperties]` лучше применять точечно, а не как массовую заглушку.

### Как использовать `#[Deprecated]`?

Для userland API, который планируется удалить или заменить. Хорошая deprecation содержит альтернативу, версию появления deprecation и план удаления. Это помогает миграциям внутри команды и внешним потребителям библиотек.

### PHP-FPM создает новый процесс на каждый запрос?

Нет. FPM держит пул worker processes. Worker обрабатывает запрос, очищает request context и переиспользуется для следующих запросов.

### Что означает shared-nothing?

Прикладное состояние запроса не разделяется с другими запросами через общий heap. Для общего состояния нужны внешние системы: БД, Redis, очередь, storage.

### Как выбрать `pm.max_children`?

По доступной памяти и реальному потреблению worker-а под нагрузкой, затем проверить CPU, DB limits и latency. Формула: FPM memory budget / p95 RSS worker.

### Что делать при `server reached pm.max_children`?

Не просто увеличивать лимит. Проверить slowlog, DB/API latency, CPU, memory, queue, timeout-и. Причина может быть в долгих запросах, а не в малом количестве воркеров.

### Что ускоряет OPcache?

Повторное использование скомпилированных opcodes из shared memory вместо чтения, парсинга и компиляции PHP-файлов на каждом запросе.

### Почему JIT обычно не ускоряет web CRUD?

Типичное web-приложение чаще упирается в БД, сеть, сериализацию, шаблонизацию, ORM и I/O. JIT помогает в CPU-bound вычислениях, но не убирает latency внешних систем и не исправляет N+1 запросы.

### Что такое copy-on-write?

При присваивании массивов/строк PHP не копирует данные сразу. Копия создается только при изменении одного из владельцев.

### Почему `foreach` по ссылке опасен?

Переменная цикла остается ссылкой на последний элемент. Если не сделать `unset($item)`, следующий цикл может случайно изменить последний элемент.

### Зачем GC, если есть reference counting?

Reference counting не освобождает циклические ссылки. GC cycle collector находит и удаляет недостижимые циклы.

### Разница между `composer install` и `composer update`?

`install` ставит версии из lock-файла. `update` пересчитывает зависимости и изменяет lock-файл.

### Нужно ли коммитить `composer.lock`?

Для приложений да, чтобы окружения были воспроизводимыми. Для библиотек чаще нет для потребителей, но lock может быть в CI самой библиотеки.

### Что делает `--classmap-authoritative`?

Composer использует только classmap и не делает fallback-поиск по файловой системе. Это быстрее, но ломает динамически появляющиеся классы, если они не попали в classmap.

### Зачем `config.platform.php`?

Чтобы резолвить зависимости под целевую версию PHP, например production, независимо от локальной версии. Это не меняет реальный runtime.

## Мини-практика

### 1. Найти проблемы миграции

```php
function findUser(string $id = null) {
    if ($id == 0) {
        return null;
    }

    $user = new User();
    $user->externalId = $id;

    return $user;
}
```

Что заметить:

- `string $id = null` в PHP 8.4 deprecated, нужно `?string $id = null`.
- `$id == 0` опасно из-за слабого сравнения.
- `$user->externalId = $id` может быть dynamic property deprecated в PHP 8.2.
- Нет return type.

Возможный вариант:

```php
function findUser(?string $id = null): ?User
{
    if ($id === null || $id === '') {
        return null;
    }

    return new User(externalId: $id);
}
```

### 2. Заменить `switch` на `match`

```php
switch ($status) {
    case '0':
    case 0:
        return 'draft';
    case 1:
        return 'published';
    default:
        return 'unknown';
}
```

Важный момент: `match` использует `===`, поэтому `'0'` и `0` - разные варианты.

```php
return match ($status) {
    '0', 0 => 'draft',
    1 => 'published',
    default => 'unknown',
};
```

### 3. Выбрать между enum и class constants

Enum лучше, если значение является частью доменной модели и нужно ограничить множество допустимых состояний:

```php
enum OrderStatus: string
{
    case Draft = 'draft';
    case Paid = 'paid';
    case Cancelled = 'cancelled';
}
```

Class constants достаточно, если это технические константы без отдельного поведения или строгого доменного смысла.

### 4. Исправить weak contract

Исходный код:

```php
function createUser($email, $age = null)
{
    if (!$email) {
        throw new Exception('Bad email');
    }
}
```

Senior-вариант:

```php
final class Email
{
    private function __construct(public readonly string $value) {}

    public static function fromString(string $value): self
    {
        if (!filter_var($value, FILTER_VALIDATE_EMAIL)) {
            throw new InvalidArgumentException('Invalid email');
        }

        return new self($value);
    }
}

function createUser(Email $email, ?int $age = null): User
{
    if ($age !== null && $age < 18) {
        throw new DomainException('User must be adult');
    }

    return new User($email, $age);
}
```

Что показать: типы описывают контракт, value object хранит invariant, exception стал конкретнее.

### 5. Заменить inheritance на composition

Плохой сигнал:

```php
class JsonFileLogger extends FileLogger
{
    protected function format(array $context): string
    {
        return json_encode($context, JSON_THROW_ON_ERROR);
    }
}
```

Лучше:

```php
interface LogFormatter
{
    /** @param array<string, mixed> $context */
    public function format(array $context): string;
}

final class JsonLogFormatter implements LogFormatter
{
    public function format(array $context): string
    {
        return json_encode($context, JSON_THROW_ON_ERROR);
    }
}

final class FileLogger
{
    public function __construct(private LogFormatter $formatter) {}
}
```

Что показать: форматирование меняется независимо от транспорта логирования.

### 6. Exception translation на boundary

```php
final class UserRepository
{
    public function save(User $user): void
    {
        try {
            $this->connection->insert('users', $user->toRow());
        } catch (Throwable $e) {
            throw new CouldNotSaveUser($user->id, previous: $e);
        }
    }
}
```

Ловить `Throwable` здесь допустимо, если это infrastructure boundary и мы не проглатываем ошибку, а сохраняем `previous`.

### 7. Property hooks для нормализации

```php
final class Product
{
    public string $sku {
        set => $this->sku = strtoupper(trim($value));
    }
}
```

Senior-ответ: hook подходит для простой локальной нормализации свойства. Если изменение SKU имеет бизнес-смысл или side effects, лучше explicit method.

### 8. Рассчитать FPM workers

Дано:

```text
RAM: 8 GB
OS/Nginx/sidecars reserve: 2 GB
p95 worker RSS: 150 MB
CPU cores: 4
```

Ответ:

```text
FPM budget = 6 GB ~= 6144 MB
pm.max_children ~= 6144 / 150 = 40
```

Если workload CPU-bound, 40 воркеров могут ухудшить latency. Нужно смотреть CPU saturation, run queue, response time, queue length и DB limits.

### 9. Найти проблему в FPM

Симптомы:

- Nginx иногда отдает 502/504.
- FPM log: `server reached pm.max_children`.
- DB CPU 95%.

Вероятное объяснение: не обязательно мало воркеров. Возможно, запросы висят на медленной БД, воркеры заняты ожиданием, очередь растет. Увеличение `pm.max_children` может добить БД. Нужно slowlog, DB slow query log, tracing, индексы, таймауты, backpressure.

### 10. Copy-on-write

```php
function touchFirst(array $data): array
{
    $data[0] = 42;
    return $data;
}

$items = range(1, 1000000);
$new = touchFirst($items);
```

При передаче в функцию копии сразу нет, но при `$data[0] = 42` произойдет separation/copy массива, потому что исходный `$items` еще ссылается на те же данные.

### 11. Composer conflict

Если `composer update vendor/package` не может поставить версию `2.0`, действия:

```bash
composer why-not vendor/package 2.0
composer why vendor/conflicting-package
composer outdated --direct
```

Дальше решить: обновить зависимый пакет, расширить constraint, заменить пакет, временно остаться на старой версии.

## Финальный чеклист

- Могу объяснить `strict_types` с примером caller/callee.
- Могу отличить `union`, `intersection`, `DNF` и не злоупотреблять ими.
- Могу объяснить variance через LSP.
- Могу написать PHPDoc generic collection/factory.
- Могу аргументировать interface vs abstract class vs trait vs composition.
- Могу описать `Throwable`, `Exception`, `Error`, `TypeError`, `ValueError`.
- Могу показать runtime use case для attributes.
- Могу объяснить enum `from()` vs `tryFrom()`.
- Могу объяснить shallow nature of `readonly`.
- Могу объяснить пользу property hooks и asymmetric visibility в PHP 8.4.
- Умею объяснить путь HTTP-запроса от Nginx до PHP-кода и обратно.
- Понимаю разницу process lifetime и request lifetime.
- Могу рассчитать `pm.max_children` и объяснить ограничения формулы.
- Знаю, как включить и читать FPM slowlog.
- Могу объяснить OPcache, preloading и deploy implications.
- Могу честно сказать, когда JIT не поможет.
- Понимаю zval/refcount/copy-on-write на практическом уровне.
- Знаю типичные memory leak patterns в long-running workers.
- Отличаю `composer install` от `update` и умею дебажить conflicts.
- Понимаю autoload optimization и риски `classmap-authoritative`.
- Учитываю Composer security: private packages, scripts, plugins, audit, tokens.
