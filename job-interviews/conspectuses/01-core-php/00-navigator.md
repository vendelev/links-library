# Core PHP: навигатор подготовки Senior/Lead

Цель блока: быстро повторить базу PHP для собеседований Senior/Lead PHP-разработчика без дублирования тем между конспектами.

Фокус не на перечислении фич ради фич, а на зрелом владении платформой: язык, типизация, runtime, производительность, миграции, Composer, качество кода и практические trade-off-ы.

## Как проходить

Если есть 1 день:

1. Пройти `01-php-74-85-versions.md`: понять ключевые изменения PHP 7.4-8.5 и опасные breaking changes.
2. Пройти `02-type-system-oop.md`: типизация, OOP, exceptions, attributes, enums, readonly, property hooks.
3. Пройти `03-runtime-fpm-opcache-composer.md`: request lifecycle, FPM, OPcache, JIT, память, GC, Composer.
4. Пройти `04-interview-practice.md`: проговорить короткие ответы и решить мини-практику.
5. Пройти `05-frankenphp.md`: понять FrankenPHP, worker mode, long-running PHP и production-риски.

Если времени мало, приоритет такой:

1. PHP 8.0-8.5 features и breaking changes.
2. `strict_types`, union/intersection/DNF, variance, PHPDoc generics.
3. `readonly`, enums, attributes, property hooks, asymmetric visibility.
4. `Throwable`, `Exception`, `Error`, `TypeError`, `ValueError`.
5. PHP-FPM lifecycle, shared-nothing, long-running workers.
6. OPcache, preloading, JIT.
7. Copy-on-write, references, GC, memory leaks.
8. Composer install/update, dependency resolution, autoload, platform config, private packages, scripts/plugins security.
9. FrankenPHP classic/worker mode, state persistence, Laravel Octane/Symfony Runtime, production observability.

## Главные темы для Lead PHP

Нужно уметь объяснить:

- как мигрировать большой legacy-монолит с PHP 7.4 на 8.3/8.4/8.5;
- как внедрять PHPStan/Psalm без остановки разработки;
- почему PHP-FPM worker живет дольше одного запроса, но request state нельзя хранить в памяти процесса;
- почему JIT редко ускоряет обычный Laravel/Symfony CRUD;
- чем enum отличается от class constants и когда enum не подходит;
- почему `readonly` не означает deep immutability;
- чем `match` отличается от `switch`;
- почему named arguments делают имена параметров частью public API;
- как dynamic properties deprecation влияет на legacy DTO/ORM/hydrator-ы;
- как исправлять implicit nullable в PHP 8.4;
- как выбирать `pm.max_children` и почему нельзя просто увеличивать воркеры;
- как Composer решает зависимости и почему `composer update` нельзя запускать на production;
- как проводить mechanical migration отдельно от refactoring.

## Карта файлов

`01-php-74-85-versions.md`:

- PHP 7.4: typed properties, arrow functions, `??=`, array unpacking, variance, WeakReference, preloading, FFI.
- PHP 8.0: union types, named arguments, attributes, constructor promotion, `match`, nullsafe, `mixed`, `static`, throw expression, string helpers, JIT, saner comparisons, `TypeError`/`ValueError`.
- PHP 8.1: enums, readonly properties, Fibers, intersection types, `never`, first-class callables, array unpacking string keys, new in initializers, final class constants, `array_is_list()`, `fsync`, `fdatasync`.
- PHP 8.2: readonly classes, DNF, standalone `null`/`false`/`true`, dynamic properties deprecated, `#[SensitiveParameter]`, Random extension, `mysqli_execute_query`.
- PHP 8.3: typed class constants, `#[Override]`, dynamic class constant fetch, `json_validate()`, readonly cloning, Randomizer additions, Date/Time exceptions.
- PHP 8.4: property hooks, asymmetric visibility, lazy objects, `#[Deprecated]`, new DOM API, array helpers, PDO subclasses, `new Foo()->bar()`, BCMath object API, mbstring helpers, implicit nullable deprecated.
- PHP 8.5: URI extension, pipe operator, clone with, `#[NoDiscard]`, closures in constant expressions, persistent cURL share handles, `array_first`, `array_last`, fatal backtraces, attributes on constants, `#[Override]` for properties, deprecated backticks/casts/old serialization.

`02-type-system-oop.md`:

- Type declarations, `strict_types`, nullable, union, intersection, DNF.
- `void`, `never`, `mixed`, `object`.
- Variance: covariance, contravariance, invariance.
- PHPDoc generics через PHPStan/Psalm.
- OOP: visibility, inheritance, interfaces, traits, composition.
- Exceptions/errors model.
- Attributes, enums, readonly properties/classes, property hooks, asymmetric visibility.

`03-runtime-fpm-opcache-composer.md`:

- Request lifecycle, shared-nothing, SAPI.
- PHP-FPM master/worker/pool, process manager modes, `pm.max_children`, slowlog, timeouts.
- OPcache, preloading, JIT.
- Memory model, zval, refcount, copy-on-write, references, arrays/strings/objects.
- GC, cycles, long-running workers.
- Composer dependency resolution, `install`/`update`, lock-файл, SemVer, constraints, autoload, platform config, private packages, scripts/plugins security.

`04-interview-practice.md`:

- Вопросы для самопроверки.
- Короткие senior-level ответы.
- Мини-практика по миграции, `match`, enum, weak contracts, composition, exception translation, FPM workers, Composer conflicts.
- Финальный чеклист.

`05-frankenphp.md`:

- FrankenPHP как PHP application server на базе Caddy.
- Classic mode, worker mode и отличие от PHP-FPM.
- Long-running PHP: state persistence, static/global/singleton pitfalls, cleanup.
- Конфигурация Caddyfile, Docker, production-чеклист.
- Laravel Octane, Symfony Runtime, observability, performance trade-off-ы.

## Основные источники

- [PHP Manual: Migration guides](https://www.php.net/manual/en/appendices.php)
- [PHP releases](https://www.php.net/releases/)
- [PHP RFCs](https://wiki.php.net/rfc)
- [PHP.Watch: PHP versions](https://php.watch/versions)
- [PHP Internals Book](https://www.phpinternalsbook.com/)
- [Composer documentation](https://getcomposer.org/doc/)
- [PHPStan documentation](https://phpstan.org/user-guide/getting-started)
- [Psalm documentation](https://psalm.dev/docs/)
- [FrankenPHP documentation](https://frankenphp.dev/docs/)

## Что перенесено из исходных файлов

Этот блок собран на основе четырех исходных конспектов:

- `job-interviews/Подготовка к собеседованию Lead PHP - PHP 7.4-8.5.md`
- `job-interviews/PHP 8.0-8.5 - features, breaking changes и вопросы.md`
- `job-interviews/PHP type system, OOP, exceptions, attributes, enums, readonly, property hooks.md`
- `job-interviews/PHP runtime, FPM, OPcache, JIT, memory, GC, Composer.md`

Повторы убраны так: обзорные списки остались в навигаторе, версионные детали в `01`, языковые объяснения в `02`, runtime/Composer в `03`, вопросы и упражнения в `04`.
