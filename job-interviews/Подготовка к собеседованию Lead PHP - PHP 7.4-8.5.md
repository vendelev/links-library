# Подготовка к собеседованию Lead PHP: PHP 7.4-8.5

Для позиции ведущего PHP-разработчика стоит готовиться не как к вопросу "что нового в PHP", 
а как к разговору про зрелое владение платформой: язык, runtime, производительность, архитектура, миграции, качество кода и управление командой.

## Главный фокус

Повторить в первую очередь:

1. PHP 8.x language features и backward compatibility.
2. Типизация: `strict_types`, union/intersection/DNF types, `readonly`, variance, generics через PHPStan/Psalm.
3. Runtime: PHP-FPM, OPcache, JIT, preloading, memory model, GC, lifecycle request-response.
4. Composer: autoload, dependency resolution, semantic versioning, private packages.
5. Ошибки и исключения: `Throwable`, `Error`, `Exception`, `ValueError`, deprecations.
6. Производительность: профилирование, N+1, кеширование, очереди, ClickHouse/PostgreSQL, Redis.
7. Асинхронный PHP: Fibers, Swoole/RoadRunner/Revolt/Amp/ReactPHP, ограничения shared-nothing модели.
8. Laravel/Symfony internals: DI container, middleware, events, queues, validation, ORM pitfalls.
9. Архитектура: Clean Architecture, CQRS, SOA, Strangler Fig, модульность, границы сервисов.
10. Code quality: PHPStan/Psalm, Rector, CodeSniffer, тесты, CI/CD, code review.

## PHP 7.4

Официальные ссылки php.net:

- [Миграция с PHP 7.3.x на PHP 7.4.x](https://www.php.net/manual/ru/migration74.php)
- [Новая функциональность PHP 7.4](https://www.php.net/manual/ru/migration74.new-features.php)
- [Изменения, которые ломают обратную совместимость](https://www.php.net/manual/ru/migration74.incompatible.php)
- [Функциональность, объявленная устаревшей в PHP 7.4](https://www.php.net/manual/ru/migration74.deprecated.php)

| Тема | Что знать |
|---|---|
| Typed properties | `public int $id;`, состояние uninitialized, отличие от `null` |
| Arrow functions | `fn($x) => $x * 2`, автоматический capture by value |
| Null coalescing assignment | `$data['key'] ??= 'default';` |
| Array unpacking | `[...]` для массивов, в 7.4 только числовые ключи |
| Covariance/contravariance | Более гибкие return/param types в наследовании |
| WeakReference | Полезно знать концептуально для кешей/объектных графов |
| OPcache preloading | Предзагрузка классов, trade-off с deploy/reload |
| FFI | Вызов C-библиотек из PHP, обычно не для web-приложений |

## PHP 8.0

Самый важный скачок после 7.4.

Официальные ссылки php.net:

- [PHP 8.0: русское описание релиза](https://www.php.net/releases/8.0/ru.php)
- [Миграция с PHP 7.4.x на PHP 8.0.x](https://www.php.net/manual/ru/migration80.php)
- [Новая функциональность PHP 8.0](https://www.php.net/manual/ru/migration80.new-features.php)
- [Изменения, которые ломают обратную совместимость](https://www.php.net/manual/ru/migration80.incompatible.php)
- [Функциональность, объявленная устаревшей в PHP 8.0](https://www.php.net/manual/ru/migration80.deprecated.php)

| Тема | Что знать |
|---|---|
| Union types | `int|string`, ограничения, отличие от PHPDoc |
| Named arguments | Удобство и риск при изменении имен параметров public API |
| Attributes | Нативные аннотации: роутинг, ORM, validation, DI |
| Constructor property promotion | Меньше boilerplate в DTO/value objects |
| `match` | Strict comparison, expression, exhaustive branches |
| Nullsafe operator | `$user?->profile?->email` |
| `mixed` | Явный "любой тип", лучше чем отсутствие типа |
| `static` return type | Fluent interfaces, named constructors |
| `throw` expression | Можно использовать в `??`, ternary, arrow functions |
| `str_contains`, `str_starts_with`, `str_ends_with` | Замена частым `strpos`-паттернам |
| JIT | Когда почти не помогает web CRUD, когда может помочь CPU-bound задачам |
| Saner comparisons | Меньше странностей при сравнении строк и чисел |
| Internal functions TypeError/ValueError | Многие ошибки стали исключениями, а не warning/null |

## PHP 8.1

Очень собеседовательная версия.

Официальные ссылки php.net:

- [PHP 8.1: русское описание релиза](https://www.php.net/releases/8.1/ru.php)
- [Миграция с PHP 8.0.x на PHP 8.1.x](https://www.php.net/manual/ru/migration81.php)
- [Новая функциональность PHP 8.1](https://www.php.net/manual/ru/migration81.new-features.php)
- [Изменения, которые ломают обратную совместимость](https://www.php.net/manual/ru/migration81.incompatible.php)
- [Функциональность, объявленная устаревшей в PHP 8.1](https://www.php.net/manual/ru/migration81.deprecated.php)

| Тема | Что знать |
|---|---|
| Enums | `enum Status: string`, отличие от class constants |
| Readonly properties | Можно записать один раз, удобно для DTO/value objects |
| Fibers | База для async frameworks, cooperative multitasking |
| Intersection types | `Countable&IteratorAggregate` |
| `never` | Функция не возвращает управление |
| First-class callable syntax | `$callable = strlen(...);` |
| Array unpacking string keys | `['a' => 1, ...$other]` |
| New in initializers | `new Foo()` в default values/attributes |
| `final` class constants | Контроль наследования констант |
| `array_is_list()` | Проверка list-like массива |
| `fsync`, `fdatasync` | Низкоуровнево, но полезно знать существование |

## PHP 8.2

Важно для современных проектов.

Официальные ссылки php.net:

- [PHP 8.2: русское описание релиза](https://www.php.net/releases/8.2/ru.php)
- [Миграция с PHP 8.1.x на PHP 8.2.x](https://www.php.net/manual/ru/migration82.php)
- [Новая функциональность PHP 8.2](https://www.php.net/manual/ru/migration82.new-features.php)
- [Изменения, которые ломают обратную совместимость](https://www.php.net/manual/ru/migration82.incompatible.php)
- [Функциональность, объявленная устаревшей в PHP 8.2](https://www.php.net/manual/ru/migration82.deprecated.php)

| Тема | Что знать |
|---|---|
| Readonly classes | Все свойства readonly, удобно для immutable DTO |
| DNF types | `(A&B)|C` |
| Standalone `null`, `false`, `true` types | Например `function foo(): false` |
| Dynamic properties deprecated | `$obj->foo = 1` без объявления вызывает deprecation |
| `#[SensitiveParameter]` | Скрытие паролей/токенов в stack trace |
| Random extension | `Random\Randomizer`, предсказуемые/безопасные генераторы |
| `mysqli_execute_query` | Менее важно, если основной стек PDO/ORM |

Особенно стоит уметь объяснить dynamic properties deprecation, потому что это часто всплывает при миграции legacy-кода.

## PHP 8.3

Менее громкая, но полезная версия.

Официальные ссылки php.net:

- [PHP 8.3: русское описание релиза](https://www.php.net/releases/8.3/ru.php)
- [Миграция с PHP 8.2.x на PHP 8.3.x](https://www.php.net/manual/ru/migration83.php)
- [Новая функциональность PHP 8.3](https://www.php.net/manual/ru/migration83.new-features.php)
- [Изменения, которые ломают обратную совместимость](https://www.php.net/manual/ru/migration83.incompatible.php)
- [Функциональность, объявленная устаревшей в PHP 8.3](https://www.php.net/manual/ru/migration83.deprecated.php)

| Тема | Что знать |
|---|---|
| Typed class constants | `public const string NAME = 'x';` |
| `#[Override]` | Защита от ошибок при переопределении методов |
| Dynamic class constant fetch | `SomeClass::{$constantName}` |
| `json_validate()` | Быстрая проверка JSON без полного decode |
| Deep-cloning readonly properties | Улучшения для `readonly` объектов |
| Randomizer additions | Новые методы для случайных значений |
| Date/Time exceptions | Более строгая модель ошибок |

## PHP 8.4

Очень важная версия для собеседований в 2026.

Официальные ссылки php.net:

- [PHP 8.4: русское описание релиза](https://www.php.net/releases/8.4/ru.php)
- [Миграция с PHP 8.3.x на PHP 8.4.x](https://www.php.net/manual/ru/migration84.php)
- [Новая функциональность PHP 8.4](https://www.php.net/manual/ru/migration84.new-features.php)
- [Изменения, которые ломают обратную совместимость](https://www.php.net/manual/ru/migration84.incompatible.php)
- [Функциональность, объявленная устаревшей в PHP 8.4](https://www.php.net/manual/ru/migration84.deprecated.php)

| Тема | Что знать |
|---|---|
| Property hooks | Нативные `get`/`set` для свойств, computed properties |
| Asymmetric visibility | Например `public private(set) string $name` |
| Lazy objects | Важно для ORM, DI container, proxy objects |
| `#[Deprecated]` | Нативная deprecation-разметка userland API |
| New DOM API + HTML5 | `Dom\HTMLDocument`, `querySelector`, `classList` |
| `array_find`, `array_find_key`, `array_any`, `array_all` | Частые utility-функции |
| PDO driver subclasses | `Pdo\Pgsql`, `Pdo\Mysql`, `PDO::connect()` |
| `new Foo()->bar()` | Без скобок вокруг `new` |
| BCMath object API | `BcMath\Number`, операторы для high precision |
| `mb_trim`, `mb_ucfirst`, `mb_lcfirst` | Наконец-то multibyte-варианты |
| Implicit nullable deprecated | `function foo(string $x = null)` надо заменить на `?string $x = null` |

Для тебя особенно релевантны property hooks, asymmetric visibility, lazy objects, PDO subclasses и deprecation implicit nullable, потому что это связано с архитектурой, DTO, ORM, миграциями и качеством кода.

## PHP 8.5

PHP 8.5 вышел 20 ноября 2025. На собеседовании достаточно знать ключевые вещи.

Официальные ссылки php.net:

- [PHP 8.5: русское описание релиза](https://www.php.net/releases/8.5/ru.php)
- [Миграция с PHP 8.4.x на PHP 8.5.x](https://www.php.net/manual/ru/migration85.php)
- [Новая функциональность PHP 8.5](https://www.php.net/manual/ru/migration85.new-features.php)
- [Изменения, которые ломают обратную совместимость](https://www.php.net/manual/ru/migration85.incompatible.php)
- [Функциональность, объявленная устаревшей в PHP 8.5](https://www.php.net/manual/ru/migration85.deprecated.php)

| Тема | Что знать |
|---|---|
| URI extension | Нативная работа с URI/URL по RFC 3986 и WHATWG URL |
| Pipe operator | `$value |> trim(...) |> strtolower(...)` |
| Clone with | `clone($object, ['property' => $value])`, удобно для immutable/readonly |
| `#[NoDiscard]` | Warning, если важный return value проигнорирован |
| Closures in constant expressions | Можно использовать static closures/callables в атрибутах/default values |
| Persistent cURL share handles | Повторное использование DNS/connect state между запросами |
| `array_first`, `array_last` | Получение первого/последнего значения массива |
| Backtrace для fatal errors | Помогает диагностике production-инцидентов |
| Attributes on constants | Расширение применения атрибутов |
| `#[Override]` для properties | Проверка переопределения свойств |
| Backtick operator deprecated | `` `ls` `` как alias для `shell_exec()` deprecated |
| Non-canonical casts deprecated | `(boolean)`, `(integer)`, `(double)` заменить на `(bool)`, `(int)`, `(float)` |
| `__sleep`/`__wakeup` soft-deprecated | Лучше `__serialize`/`__unserialize` |

## Что могут спросить именно тебя

С учетом резюме я бы ожидал такие вопросы:

1. Как бы ты мигрировал большой legacy-монолит с PHP 7.4 на 8.3/8.4/8.5?
2. Как внедрять PHPStan/Psalm в старый проект без остановки разработки?
3. Как проектировать сервис при выносе из монолита через Strangler Fig?
4. Где граница между модулем, composer-пакетом, микросервисом и bounded context?
5. Почему CQRS не всегда нужен?
6. Как диагностировать медленный endpoint в Laravel/Symfony/CakePHP?
7. Как устроен PHP-FPM lifecycle и почему in-memory state между запросами обычно не живет?
8. Когда Redis cache помогает, а когда маскирует проблему?
9. Как проектировать очереди: идемпотентность, retry, dead letter queue, ordering, visibility timeout?
10. Как обеспечить обратную совместимость REST API/OpenAPI?
11. Как правильно хранить историю: PostgreSQL vs ClickHouse vs S3/MinIO?
12. Как организовать code review для команды, чтобы он не стал bottleneck?
13. Как ты принимаешь архитектурные решения и фиксируешь их: ADR, RFC, docs?
14. Как проводить собеседование PHP-разработчика уровня middle/senior?
15. Как оценить качество чужого кода за первые 2 часа?

## Минимальный план подготовки

Если времени мало, я бы шел так:

1. За 1 день: PHP 8.0-8.5 features и breaking changes.
2. За 1 день: PHP type system, OOP, exceptions, attributes, enums, readonly, property hooks.
3. За 1 день: PHP runtime, FPM, OPcache, JIT, memory, GC, Composer.
4. За 1 день: Laravel/Symfony internals, DI, middleware, ORM, queues.
5. За 1 день: PostgreSQL/ClickHouse/Redis/RabbitMQ, индексы, транзакции, блокировки.
6. За 1 день: архитектурные кейсы из твоего опыта: монолит, SOA, CQRS, Strangler Fig.
7. За 1 день: leadership stories по STAR: конфликт, провал, найм, code review, production incident.

## Особенно повторить по PHP

Если выбрать только самое собеседовательное:

- `readonly`, enums, attributes, property hooks, asymmetric visibility.
- `mixed`, `never`, union/intersection/DNF types.
- Variance: covariance/contravariance.
- `strict_types=1`: что реально делает, а что нет.
- `==` vs `===`, edge cases сравнений.
- References, copy-on-write, memory leaks, circular references, GC.
- Generators vs arrays.
- Fibers vs threads vs async event loop.
- OPcache, preloading, JIT.
- Exceptions/errors model: `Throwable`, `Error`, `Exception`, `TypeError`, `ValueError`.
- Composer autoload: PSR-4, classmap, optimized autoload.
- Static analysis: baseline, levels, generics через PHPDoc.
- Миграции версий PHP и deprecation-driven development.
