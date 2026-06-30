# PHP 7.4-8.5: features, breaking changes и миграции

Цель: повторить изменения PHP 7.4-8.5 перед собеседованием Senior/Lead PHP-разработчика. Фокус на production-коде: типизация, legacy migration, static analysis, runtime-поведение, обратная совместимость и архитектурные последствия.

## Официальные ссылки

PHP 7.4:

- [Migration from PHP 7.3.x to PHP 7.4.x](https://www.php.net/manual/en/migration74.php)
- [New features PHP 7.4](https://www.php.net/manual/en/migration74.new-features.php)
- [Backward incompatible changes PHP 7.4](https://www.php.net/manual/en/migration74.incompatible.php)
- [Deprecated features PHP 7.4](https://www.php.net/manual/en/migration74.deprecated.php)

PHP 8.0-8.5:

- [PHP 8.0 release](https://www.php.net/releases/8.0/en.php)
- [Migration PHP 7.4 to PHP 8.0](https://www.php.net/manual/en/migration80.php)
- [PHP 8.1 release](https://www.php.net/releases/8.1/en.php)
- [Migration PHP 8.0 to PHP 8.1](https://www.php.net/manual/en/migration81.php)
- [PHP 8.2 release](https://www.php.net/releases/8.2/en.php)
- [Migration PHP 8.1 to PHP 8.2](https://www.php.net/manual/en/migration82.php)
- [PHP 8.3 release](https://www.php.net/releases/8.3/en.php)
- [Migration PHP 8.2 to PHP 8.3](https://www.php.net/manual/en/migration83.php)
- [PHP 8.4 release](https://www.php.net/releases/8.4/en.php)
- [Migration PHP 8.3 to PHP 8.4](https://www.php.net/manual/en/migration84.php)
- [PHP 8.5 release](https://www.php.net/releases/8.5/en.php)
- [Migration PHP 8.4 to PHP 8.5](https://www.php.net/manual/en/migration85.php)

Дополнительные обзоры:

- [PHP.Watch: PHP 8.0](https://php.watch/versions/8.0)
- [PHP.Watch: PHP 8.1](https://php.watch/versions/8.1)
- [PHP.Watch: PHP 8.2](https://php.watch/versions/8.2)
- [PHP.Watch: PHP 8.3](https://php.watch/versions/8.3)
- [PHP.Watch: PHP 8.4](https://php.watch/versions/8.4)
- [PHP.Watch: PHP 8.5](https://php.watch/versions/8.5)

## PHP 7.4

| Тема | Что знать |
|---|---|
| Typed properties | `public int $id;`, состояние uninitialized, отличие от `null` |
| Arrow functions | `fn($x) => $x * 2`, автоматический capture by value |
| Null coalescing assignment | `$data['key'] ??= 'default';` |
| Array unpacking | `[...]` для массивов, в 7.4 только числовые ключи |
| Covariance/contravariance | Более гибкие return/param types в наследовании |
| WeakReference | Концептуально полезно для кешей и объектных графов |
| OPcache preloading | Предзагрузка классов, trade-off с deploy/reload |
| FFI | Вызов C-библиотек из PHP, обычно не для web-приложений |

Собеседовательный фокус: typed properties и uninitialized state, variance, preloading как часть OPcache/deploy-темы.

## PHP 8.0

PHP 8.0 - большой скачок в сторону строгой и выразительной модели языка.

| Фича | Зачем важно |
|---|---|
| Union types | Реальные runtime-типы вместо части PHPDoc: `int|string`, `User|null` |
| Named arguments | Удобны для длинных сигнатур, но имена параметров становятся public API |
| Attributes | Нативные metadata вместо части docblock-аннотаций: routes, ORM, validation, DI |
| Constructor property promotion | Меньше boilerplate в DTO/value objects |
| `match` | Strict comparison, expression, exhaustive branches, нет fall-through |
| Nullsafe operator | `$user?->profile?->email` |
| `mixed` | Явный любой тип вместо отсутствия типа |
| `static` return type | Fluent interfaces, named constructors |
| `throw` expression | Можно использовать в `??`, ternary, arrow functions |
| `str_contains`, `str_starts_with`, `str_ends_with` | Читабельнее и безопаснее ручного `strpos` |
| JIT | Концептуально важно, но редко ускоряет обычный web CRUD |
| Saner comparisons | Меньше странностей при сравнении строк и чисел |
| Internal functions TypeError/ValueError | Многие ошибки стали исключениями, а не warning/null |

Что ломает legacy:

- Более строгие ошибки внутренних функций: вместо warning/null часто `TypeError` или `ValueError`.
- Изменились правила сравнения строк и чисел, например `0 == 'foo'` теперь `false`.
- `match` использует `===`, поэтому это не drop-in replacement для `switch`.
- Named arguments опасны для библиотек: переименование параметра может стать breaking change.
- Многие ресурсы стали объектами: `CurlHandle`, `GdImage`, `Socket` и другие.

Senior-формулировка: при миграции с 7.4 на 8.0 я бы особенно проверял слабые сравнения, обработку warning-ов от внутренних функций, public API с named arguments и места, где код рассчитывал на нестрогое поведение PHP.

## PHP 8.1

| Фича | Зачем важно |
|---|---|
| Enums | Типобезопасная замена наборам class constants/string constants |
| Readonly properties | Запись один раз, удобно для DTO/value objects |
| Fibers | База для async frameworks и cooperative multitasking |
| Intersection types | Требование нескольких контрактов: `Countable&IteratorAggregate` |
| `never` | Функция не возвращает управление: `exit`, infinite loop, exception-only flow |
| First-class callable syntax | `$callable = strlen(...);` |
| Array unpacking string keys | Можно объединять associative arrays через `...` |
| New in initializers | `new Foo()` в default values/attributes и похожих initializer-контекстах |
| `final` class constants | Контроль наследования констант |
| `array_is_list()` | Проверка list-like массива |
| `fsync`, `fdatasync` | Низкоуровневые функции синхронизации данных на диск |

Что ломает legacy:

- `readonly` требует аккуратного проектирования: нельзя менять свойство после инициализации.
- Enums нельзя воспринимать как простые строки, даже backed enum требует обращения к `->value`.
- Fibers сами по себе не делают код асинхронным: нужен event loop/framework.
- Некоторые внутренние функции и расширения получили изменения поведения и типов.

Senior-формулировка: PHP 8.1 важен из-за enums, readonly properties и fibers. Enums подходят для доменных состояний и закрытых наборов значений, readonly - для immutable-style DTO/value objects, fibers - как фундамент async runtime вроде Revolt/Amp, ReactPHP, RoadRunner/Swoole-экосистемы.

## PHP 8.2

| Фича | Зачем важно |
|---|---|
| Readonly classes | Все instance-свойства класса readonly |
| DNF types | Более выразительные типы: `(A&B)|C` |
| Standalone `null`, `false`, `true` types | Явное описание специальных return/value types |
| Dynamic properties deprecated | Борьба с опечатками и неявными контрактами объектов |
| `#[SensitiveParameter]` | Скрытие секретов в stack trace |
| Random extension | Более явная, тестируемая и безопасная работа со случайностью |
| `mysqli_execute_query` | Упрощение mysqli API, менее важно для PDO/ORM-стеков |

Главная миграционная тема: dynamic properties deprecated. `$object->unknown = 'value'` вызывает deprecation, если свойство не объявлено.

Что проверить:

- старые ActiveRecord/ORM/DTO;
- magic models и hydrator-ы;
- `stdClass`-like объекты;
- реальные опечатки в именах свойств;
- случаи, где нужен `stdClass`, `#[AllowDynamicProperties]`, `__get`/`__set` или явные свойства.

Senior-формулировка: в legacy-проекте я бы не ставил `#[AllowDynamicProperties]` везде, а классифицировал случаи: где произвольный контейнер данных, где DTO с явными свойствами, где магия ORM, а где реальная ошибка.

## PHP 8.3

| Фича | Зачем важно |
|---|---|
| Typed class constants | `public const string NAME = 'x';` |
| `#[Override]` | Защита от ошибок при переопределении методов |
| Dynamic class constant fetch | `SomeClass::{$constantName}` |
| `json_validate()` | Проверка JSON без полного decode |
| Deep-cloning readonly properties | Удобнее работать с immutable/readonly объектами |
| Randomizer additions | Новые методы для случайных значений |
| Date/Time exceptions | Более строгая и явная модель ошибок |

Что ломает legacy:

- `#[Override]` сам по себе не ломает код, но выявляет ошибки наследования.
- Типизированные константы могут выявить несовместимость при наследовании.
- Изменения Date/Time могут проявиться там, где код ожидал warning или неявное поведение.

Senior-формулировка: PHP 8.3 не революционный, но полезен для качества кода. `#[Override]` ценен в больших кодовых базах, а `json_validate()` полезен, когда нужно проверить JSON без выделения памяти на полноценный decode.

## PHP 8.4

| Фича | Зачем важно |
|---|---|
| Property hooks | Нативные `get`/`set` для свойств, computed properties |
| Asymmetric visibility | `public private(set) string $name`, шире чтение, уже запись |
| Lazy objects | Важно для ORM, DI containers, proxy objects |
| `#[Deprecated]` | Нативная deprecation-разметка userland API |
| New DOM API + HTML5 | `Dom\HTMLDocument`, `querySelector`, `classList` |
| `array_find`, `array_find_key`, `array_any`, `array_all` | Частые операции с массивами без ручных циклов |
| PDO driver subclasses | `Pdo\Pgsql`, `Pdo\Mysql`, `PDO::connect()` |
| `new Foo()->bar()` | Вызов метода нового объекта без скобок вокруг `new` |
| BCMath object API | `BcMath\Number`, операторы для high precision |
| `mb_trim`, `mb_ucfirst`, `mb_lcfirst` | Multibyte-варианты частых string-функций |

Что ломает legacy:

- Implicit nullable deprecated: `function foo(string $x = null)` нужно заменить на `function foo(?string $x = null)`.
- Некоторые расширения вынесены из core в PECL: IMAP, OCI8, PDO_OCI, pspell.
- `E_STRICT` deprecated.
- Некоторые ошибки стали строже, например invalid mode в `round()` теперь `ValueError`.
- Изменения в `exit()` могут затронуть тесты и код, который перехватывает завершение.

Senior-формулировка: PHP 8.4 важен для проектирования моделей и DTO. Property hooks и asymmetric visibility уменьшают boilerplate вокруг getter/setter-ов, но не должны прятать тяжелую бизнес-логику за чтением свойства. При миграции я бы отдельно искал implicit nullable parameters через Rector/static analysis.

## PHP 8.5

PHP 8.5 вышел 20 ноября 2025. На собеседовании достаточно знать ключевые вещи.

| Фича | Зачем важно |
|---|---|
| URI extension | Нативная работа с URI/URL по RFC 3986 и WHATWG URL |
| Pipe operator | `$value |> trim(...) |> strtolower(...)`, преобразования слева направо |
| Clone with | `clone($object, ['property' => $value])`, with-er pattern для immutable/readonly |
| `#[NoDiscard]` | Warning, если важный return value проигнорирован |
| Closures in constant expressions | Static closures/callables в атрибутах/default values |
| Persistent cURL share handles | Повторное использование DNS/connect state между запросами |
| `array_first`, `array_last` | Получение первого/последнего значения массива |
| Backtrace для fatal errors | Помогает диагностике production-инцидентов |
| Attributes on constants | Расширение применения атрибутов |
| `#[Override]` для properties | Проверка переопределения свойств |

Что ломает legacy:

- Backtick operator deprecated: `` `ls` `` как alias для `shell_exec()`.
- Non-canonical casts deprecated: `(boolean)`, `(integer)`, `(double)`, `(binary)` заменить на `(bool)`, `(int)`, `(float)`, `(string)`.
- `disable_classes` INI setting удален.
- `case` с `;` вместо `:` deprecated.
- `null` как array offset или аргумент `array_key_exists()` deprecated.
- `__sleep()` и `__wakeup()` soft-deprecated, лучше `__serialize()` и `__unserialize()`.
- Casting `NAN` и некоторые float-to-int cases стали шумнее через warning.

Senior-формулировка: PHP 8.5 продолжает движение к более строгому и выразительному языку. URI extension закрывает проблему ненадежного парсинга URL через `parse_url()`. Pipe operator полезен для чистых преобразований данных, но не для нечитаемых цепочек с side effects. Clone with хорошо ложится на immutable DTO/value objects и readonly-классы.

## Самые опасные breaking changes

| Версия | Риск | Что делать |
|---|---|---|
| 8.0 | Internal functions чаще бросают `TypeError`/`ValueError` | Прогнать тесты, проверить обработку ошибок, добавить валидацию входных данных |
| 8.0 | Изменение сравнений строк и чисел | Искать `==`, `!=`, `switch`, приводить к строгим сравнениям |
| 8.0 | Named arguments делают имена параметров частью API | Не менять имена параметров public API без major release |
| 8.1 | Enums требуют явной работы с `->value` | Не смешивать enum object и backed value |
| 8.2 | Dynamic properties deprecated | Явные свойства, DTO, `stdClass`, `__get`/`__set`, точечный `#[AllowDynamicProperties]` |
| 8.4 | Implicit nullable deprecated | `T $x = null` заменить на `?T $x = null` |
| 8.4 | Расширения вынесены в PECL | Проверить Docker images, CI, production packages |
| 8.5 | Deprecated casts/backticks/old serialization | Rector/static analysis, grep по опасным паттернам |

## Миграция проекта с PHP 7.4 на 8.4/8.5

1. Зафиксировать текущую версию, зависимости и окружение: Docker image, extensions, Composer lock, CI.
2. Обновить зависимости до версий, которые поддерживают целевой PHP.
3. Включить статический анализ и baseline, чтобы отделить старый долг от новых ошибок.
4. Прогнать Rector с наборами для миграции PHP-версий.
5. Исправить явные deprecations: dynamic properties, implicit nullable, старые casts, несовместимые сигнатуры.
6. Убрать опасные `==` там, где есть сравнение чисел, строк, enum values, IDs, статусов.
7. Проверить PHP extensions в Docker/CI/prod: intl, mbstring, pdo_pgsql, redis, amqp, curl, soap, gd.
8. Прогнать unit/integration/e2e tests.
9. Отдельно проверить очереди, cron-команды, CLI-скрипты, миграции БД и long-running workers.
10. Включить deprecation logging на staging и собрать реальные проблемы.
11. Сделать canary/blue-green deploy, если система критичная.

## Что проговорить как Lead

- Я не обновляю PHP ради новых фич: смотрю на security support, зависимости, стоимость миграции, CI, Docker images, extensions, observability и rollback plan.
- Я не включаю новые конструкции массово без правил команды: property hooks, pipe operator и asymmetric visibility требуют coding guidelines.
- Я использую Rector и статический анализ, но не доверяю автоправкам вслепую.
- Я разделяю mechanical migration и refactoring: сначала безопасно обновляем платформу, потом улучшаем архитектуру.
- Я обязательно проверяю workers, cron, queues и CLI-команды, потому что они часто хуже покрыты тестами, чем HTTP endpoints.
