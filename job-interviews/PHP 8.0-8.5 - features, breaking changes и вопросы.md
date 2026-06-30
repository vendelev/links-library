# PHP 8.0-8.5: features, breaking changes и вопросы

Цель файла: быстро повторить изменения PHP 8.0-8.5 перед собеседованием на позицию Senior/Lead PHP-разработчика.

Фокус не на перечислении всех изменений, а на том, что чаще всего важно в реальных проектах: типизация, миграции legacy-кода, статический анализ, runtime-поведение, обратная совместимость и архитектурные последствия.

## Как проходить за 1 день

1. Сначала пройти разделы `PHP 8.0` - `PHP 8.5` и выписать незнакомые темы.
2. Потом отдельно пройти `Самые опасные breaking changes`.
3. Потом ответить вслух на вопросы из раздела `Вопросы для самопроверки`.
4. В конце подготовить 3-5 историй из своего опыта: миграция версии PHP, внедрение статического анализа, борьба с legacy, performance incident, архитектурное решение.

## Официальные ссылки php.net

- [PHP 8.0: русское описание релиза](https://www.php.net/releases/8.0/ru.php)
- [Миграция с PHP 7.4.x на PHP 8.0.x](https://www.php.net/manual/ru/migration80.php)
- [PHP 8.1: русское описание релиза](https://www.php.net/releases/8.1/ru.php)
- [Миграция с PHP 8.0.x на PHP 8.1.x](https://www.php.net/manual/ru/migration81.php)
- [PHP 8.2: русское описание релиза](https://www.php.net/releases/8.2/ru.php)
- [Миграция с PHP 8.1.x на PHP 8.2.x](https://www.php.net/manual/ru/migration82.php)
- [PHP 8.3: русское описание релиза](https://www.php.net/releases/8.3/ru.php)
- [Миграция с PHP 8.2.x на PHP 8.3.x](https://www.php.net/manual/ru/migration83.php)
- [PHP 8.4: русское описание релиза](https://www.php.net/releases/8.4/ru.php)
- [Миграция с PHP 8.3.x на PHP 8.4.x](https://www.php.net/manual/ru/migration84.php)
- [PHP 8.5: русское описание релиза](https://www.php.net/releases/8.5/ru.php)
- [Миграция с PHP 8.4.x на PHP 8.5.x](https://www.php.net/manual/ru/migration85.php)

## PHP 8.0

### Что появилось

| Фича | Зачем важно |
|---|---|
| Union types | Можно заменить часть PHPDoc реальными runtime-типами: `int|string`, `User|null` |
| Named arguments | Удобно для функций с большим количеством параметров, но имена параметров становятся частью public API |
| Attributes | Нативные метаданные вместо docblock-аннотаций: роутинг, ORM, validation, DI |
| Constructor property promotion | Меньше boilerplate в DTO/value objects |
| `match` | Строгое сравнение, expression-style, нет fall-through как в `switch` |
| Nullsafe operator | Упрощает цепочки nullable-вызовов: `$user?->profile?->email` |
| `mixed` | Явное обозначение "любой тип" вместо отсутствия типа |
| `static` return type | Удобно для fluent API и named constructors |
| `throw` expression | Можно использовать в `??`, ternary, arrow functions |
| `str_contains`, `str_starts_with`, `str_ends_with` | Безопаснее и читабельнее, чем ручной `strpos` |
| JIT | Важен концептуально, но обычно не дает большого выигрыша для типичных web CRUD |

### Что ломает legacy

- Более строгие ошибки внутренних функций: вместо warning/null часто `TypeError` или `ValueError`.
- Изменились правила сравнения строк и чисел: например `0 == 'foo'` теперь `false`.
- `match` использует строгое сравнение, это не drop-in replacement для `switch`.
- Named arguments опасны для библиотек: переименование параметра может стать breaking change.
- Многие ресурсы стали объектами: CurlHandle, GdImage, Socket и другие.

### Что ответить на собеседовании

PHP 8.0 усилил типизацию и предсказуемость языка. Самые важные изменения для production-кода: union types, attributes, named arguments, constructor promotion, `match`, nullsafe operator и более строгая модель ошибок. При миграции с 7.4 я бы особенно проверял сравнения `==`, обработку warning-ов от внутренних функций и места, где код рассчитывал на нестрогое поведение PHP.

## PHP 8.1

### Что появилось

| Фича | Зачем важно |
|---|---|
| Enums | Типобезопасная замена наборам class constants/string constants |
| Readonly properties | Можно записать значение один раз, удобно для DTO/value objects |
| Fibers | База для async frameworks и cooperative multitasking |
| Intersection types | Требование одновременно нескольких контрактов: `Countable&IteratorAggregate` |
| `never` | Функция не возвращает управление: `exit`, infinite loop, exception-only flow |
| First-class callable syntax | `$callable = strlen(...);` |
| Array unpacking string keys | Можно объединять associative arrays через `...` |
| `array_is_list()` | Проверка, является ли массив list-like |
| New in initializers | Можно использовать `new` в некоторых initializer-контекстах |

### Что ломает legacy

- `readonly` требует аккуратного проектирования: нельзя менять свойство после инициализации.
- Enums нельзя воспринимать как простые строки, даже backed enum требует обращения к `->value`.
- Fibers сами по себе не делают код асинхронным: нужен event loop/framework.
- Некоторые внутренние функции и расширения получили изменения поведения и типов.

### Что ответить на собеседовании

PHP 8.1 важен из-за enums, readonly properties и fibers. Enums я бы использовал для доменных состояний и ограниченных наборов значений. Readonly хорошо подходит для immutable DTO и value objects. Fibers важны не как инструмент для ежедневного ручного использования, а как фундамент для асинхронных библиотек и runtime вроде Revolt/Amp, ReactPHP, RoadRunner/Swoole-экосистемы.

## PHP 8.2

### Что появилось

| Фича | Зачем важно |
|---|---|
| Readonly classes | Все instance-свойства класса readonly |
| DNF types | Более выразительные типы: `(A&B)|C` |
| Standalone `null`, `false`, `true` types | Можно явно описывать специальные return-типы |
| Dynamic properties deprecated | Борьба с ошибками в именах свойств и магическим legacy-кодом |
| `#[SensitiveParameter]` | Скрытие секретов в stack trace |
| Random extension | Более явная и тестируемая работа со случайностью |

### Что ломает legacy

- Dynamic properties deprecated: `$object->unknown = 'value'` вызывает deprecation, если свойство не объявлено.
- Это особенно влияет на старые ActiveRecord/ORM/DTO, магические модели, hydrator-ы и stdClass-like объекты.
- Нужно различать настоящие dynamic properties, `stdClass`, `#[AllowDynamicProperties]`, `__get`/`__set` и явные свойства.

### Что ответить на собеседовании

Главная миграционная тема PHP 8.2 - deprecation dynamic properties. Это полезное изменение, потому что оно ловит опечатки и неявные контракты объектов. В legacy-проекте я бы не ставил `#[AllowDynamicProperties]` везде, а сначала классифицировал случаи: где нужен `stdClass`, где нужен DTO с явными свойствами, где магия ORM, а где реальная ошибка.

## PHP 8.3

### Что появилось

| Фича | Зачем важно |
|---|---|
| Typed class constants | Константы класса получают тип: `public const string NAME = 'x';` |
| `#[Override]` | Защита от ошибок при переопределении методов |
| Dynamic class constant fetch | `SomeClass::{$constantName}` |
| `json_validate()` | Проверка JSON без полного decode |
| Улучшения readonly cloning | Удобнее работать с immutable-объектами |
| Date/Time exceptions | Более строгая и явная обработка ошибок |

### Что ломает legacy

- `#[Override]` сам по себе не ломает код, но выявляет ошибки наследования.
- Типизированные константы могут выявить несовместимость при наследовании.
- Изменения в Date/Time могут проявиться там, где код ожидал warning или неявное поведение.

### Что ответить на собеседовании

PHP 8.3 не такой революционный, как 8.0 или 8.1, но полезен для качества кода. `#[Override]` особенно ценен в больших кодовых базах: он позволяет явно сказать, что метод должен переопределять родительский, и получить ошибку, если контракт сломался. `json_validate()` полезен, когда нужно проверить JSON без выделения памяти на полноценный decode.

## PHP 8.4

### Что появилось

| Фича | Зачем важно |
|---|---|
| Property hooks | Нативные `get`/`set` для свойств, computed properties |
| Asymmetric visibility | Можно читать публично, а писать только из private/protected scope |
| Lazy objects | Важно для ORM, DI containers, proxy objects |
| `#[Deprecated]` | Нативная deprecation-разметка userland API |
| New DOM API + HTML5 | Современная работа с HTML5 DOM |
| `array_find`, `array_find_key`, `array_any`, `array_all` | Частые операции с массивами без ручных циклов |
| PDO driver subclasses | Более точные классы подключений: `Pdo\Pgsql`, `Pdo\Mysql` |
| `new Foo()->bar()` | Можно обращаться к методу нового объекта без скобок |
| BCMath object API | Объектная работа с числами произвольной точности |
| `mb_trim`, `mb_ucfirst`, `mb_lcfirst` | Multibyte-варианты частых string-функций |

### Что ломает legacy

- Implicit nullable deprecated: `function foo(string $x = null)` нужно заменить на `function foo(?string $x = null)`.
- Некоторые расширения вынесены из core в PECL: IMAP, OCI8, PDO_OCI, pspell.
- `E_STRICT` deprecated.
- Некоторые ошибки стали строже, например invalid mode в `round()` теперь `ValueError`.
- Изменения в `exit()` могут затронуть тесты и код, который перехватывает поведение завершения.

### Что ответить на собеседовании

PHP 8.4 важен для проектирования моделей и DTO. Property hooks и asymmetric visibility уменьшают boilerplate вокруг getter/setter-ов, но требуют аккуратности, чтобы не спрятать сложную бизнес-логику в свойства. Lazy objects важны для ORM и DI, потому что позволяют лениво инициализировать тяжелые зависимости и proxy. При миграции я бы отдельно искал implicit nullable parameters, потому что это массовая и хорошо автоматизируемая проблема через Rector/static analysis.

## PHP 8.5

### Что появилось

| Фича | Зачем важно |
|---|---|
| URI extension | Нативная работа с URI/URL по RFC 3986 и WHATWG URL |
| Pipe operator | Читабельные цепочки преобразований слева направо |
| Clone with | Удобный with-er pattern для immutable/readonly объектов |
| `#[NoDiscard]` | Warning, если важный return value проигнорирован |
| Closures in constant expressions | Static closures/callables в атрибутах/default values |
| Persistent cURL share handles | Повторное использование DNS/connect state между запросами |
| `array_first`, `array_last` | Простое получение первого/последнего значения массива |
| Fatal errors with backtrace | Лучше диагностика production-инцидентов |

### Что ломает legacy

- Backtick operator deprecated: `` `ls` `` больше не стоит использовать как alias для `shell_exec()`.
- Non-canonical casts deprecated: `(boolean)`, `(integer)`, `(double)`, `(binary)` заменить на `(bool)`, `(int)`, `(float)`, `(string)`.
- `disable_classes` INI setting удален.
- `case` с `;` вместо `:` deprecated.
- `null` как array offset или аргумент `array_key_exists()` deprecated.
- `__sleep()` и `__wakeup()` soft-deprecated, лучше `__serialize()` и `__unserialize()`.
- Casting `NAN` и некоторые float-to-int cases стали шумнее через warning.

### Что ответить на собеседовании

PHP 8.5 продолжает движение к более строгому и выразительному языку. URI extension закрывает старую проблему ненадежного парсинга URL через `parse_url()`. Pipe operator полезен для чистых преобразований данных, но его не стоит превращать в нечитаемые цепочки с побочными эффектами. Clone with хорошо ложится на immutable DTO/value objects и readonly-классы. В миграции я бы проверял deprecated casts, backticks, `array_key_exists(null, ...)`, старую сериализацию и нестандартный синтаксис `case`.

## Самые опасные breaking changes при миграции

| Версия | Риск | Что делать |
|---|---|---|
| 8.0 | Internal functions стали чаще бросать `TypeError`/`ValueError` | Прогнать тесты, проверить обработку ошибок, добавить валидацию входных данных |
| 8.0 | Изменение сравнений строк и чисел | Искать `==`, `!=`, `switch`, приводить к строгим сравнениям |
| 8.0 | Named arguments делают имена параметров частью API | Не менять имена параметров в публичных библиотеках без major release |
| 8.1 | Enums требуют явной работы с `->value` | Не смешивать enum object и backed value |
| 8.2 | Dynamic properties deprecated | Явные свойства, DTO, `stdClass`, `__get`/`__set`, точечный `#[AllowDynamicProperties]` |
| 8.4 | Implicit nullable deprecated | `T $x = null` заменить на `?T $x = null` |
| 8.4 | Расширения вынесены в PECL | Проверить Docker images, CI, production packages |
| 8.5 | Deprecated old casts/backticks/old serialization | Rector/static analysis, grep по опасным паттернам |

## Как бы я мигрировал проект с PHP 7.4 на 8.4/8.5

1. Зафиксировал текущую версию, зависимости и окружение: Docker image, extensions, Composer lock, CI.
2. Обновил зависимости до версий, которые поддерживают целевой PHP.
3. Включил статический анализ и baseline, чтобы отделить старый долг от новых ошибок.
4. Прогнал Rector с наборами для миграции PHP-версий.
5. Исправил явные deprecations: dynamic properties, implicit nullable, старые casts, несовместимые сигнатуры.
6. Убрал опасные `==` там, где есть сравнение чисел, строк, enum values, IDs, статусов.
7. Проверил расширения PHP в Docker/CI/prod: intl, mbstring, pdo_pgsql, redis, amqp, curl, soap, gd и т.п.
8. Прогнал unit/integration/e2e tests.
9. Отдельно проверил очереди, cron-команды, CLI-скрипты, миграции БД и long-running workers.
10. Включил deprecation logging на staging и собрал реальные проблемы.
11. Сделал canary/blue-green deploy, если система критичная.

## Вопросы для самопроверки

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

## Короткие ответы

### Чем `match` отличается от `switch`?

`match` - это expression, возвращает значение, использует строгое сравнение `===`, не имеет fall-through и требует явного покрытия вариантов или `default`. `switch` - statement, использует слабое сравнение `==` и может проваливаться между case без `break`.

### Почему named arguments могут сломать BC?

Потому что имя параметра становится частью публичного контракта. Если пользователь вызывает `foo(timeout: 10)`, то переименование `$timeout` в `$requestTimeout` ломает его код, даже если порядок и тип параметра не изменились.

### Что такое `readonly` и почему это не deep immutability?

`readonly` запрещает повторно присвоить свойство после инициализации. Но если свойство содержит объект, внутреннее состояние этого объекта может быть изменяемым. Поэтому `readonly` защищает ссылку, но не всегда весь объектный граф.

### Что такое Fibers?

Fibers - механизм cooperative multitasking: выполнение можно приостановить и продолжить позже. Это не потоки и не параллельное выполнение CPU-кода. Fibers полезны как низкоуровневая база для async runtimes, event loop и неблокирующего I/O.

### Почему JIT обычно не ускоряет web CRUD?

Типичное web-приложение чаще упирается в БД, сеть, сериализацию, шаблонизацию, ORM и I/O. JIT помогает в CPU-bound вычислениях, но не убирает latency внешних систем и не исправляет N+1 запросы.

### Как относиться к property hooks?

Это удобный способ выразить computed или validated properties без ручных getter/setter-ов. Но есть риск спрятать тяжелую или неожиданную бизнес-логику за обычным чтением/записью свойства. Для доменной логики по-прежнему лучше явные методы.

### Как исправлять dynamic properties?

Сначала понять назначение объекта. Если это произвольный контейнер данных - возможно, нужен `stdClass` или массив. Если это DTO - объявить свойства явно. Если это ORM/proxy - использовать поддерживаемый механизм фреймворка. `#[AllowDynamicProperties]` лучше применять точечно, а не как массовую заглушку.

### Как использовать `#[Deprecated]`?

Для userland API, который планируется удалить или заменить. Хорошая deprecation должна содержать альтернативу, версию появления deprecation и план удаления. Это помогает миграциям внутри команды и внешним потребителям библиотек.

## Мини-практика

### Задача 1

Найти потенциальные проблемы миграции:

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

### Задача 2

Заменить `switch` на `match`, но не потерять поведение:

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

### Задача 3

Выбрать между enum и class constants.

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

## Что стоит проговорить как Lead

- Я не обновляю PHP "ради новых фич". Я смотрю на security support, зависимости, стоимость миграции, CI, Docker images, extensions, observability и rollback plan.
- Я не включаю новые конструкции массово без правил команды. Например, property hooks, pipe operator и asymmetric visibility требуют coding guidelines.
- Я использую Rector и статический анализ, но не доверяю автоправкам вслепую.
- Я разделяю mechanical migration и refactoring. Сначала безопасно обновляем платформу, потом улучшаем архитектуру.
- Я обязательно проверяю workers, cron, queues и CLI-команды, потому что они часто хуже покрыты тестами, чем HTTP endpoints.
