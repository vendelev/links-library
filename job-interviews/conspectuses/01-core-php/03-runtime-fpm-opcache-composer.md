# PHP runtime, FPM, OPcache, JIT, memory, GC, Composer

Цель: системно понимать PHP runtime для собеседования Senior/Lead PHP-разработчика. Фокус на практических объяснениях, типичных ловушках, диагностике и production trade-off-ах.

## Документация и статьи

PHP runtime и SAPI:

- [PHP Manual: Installation and Configuration](https://www.php.net/manual/en/install.php)
- [PHP Manual: Command line usage](https://www.php.net/manual/en/features.commandline.php)
- [PHP Manual: Runtime configuration](https://www.php.net/manual/en/configuration.php)
- [PHP Manual: php.ini directives](https://www.php.net/manual/en/ini.core.php)

PHP-FPM:

- [PHP Manual: FastCGI Process Manager](https://www.php.net/manual/en/install.fpm.php)
- [PHP Manual: FPM configuration](https://www.php.net/manual/en/install.fpm.configuration.php)
- [PHP Manual: FPM status page](https://www.php.net/manual/en/fpm.status.php)
- [Tideways: An introduction to PHP-FPM tuning](https://tideways.com/profiler/blog/an-introduction-to-php-fpm-tuning)

OPcache, preloading, JIT:

- [PHP Manual: OPcache](https://www.php.net/manual/en/book.opcache.php)
- [PHP Manual: OPcache configuration](https://www.php.net/manual/en/opcache.configuration.php)
- [PHP Manual: Preloading](https://www.php.net/manual/en/opcache.preloading.php)
- [PHP Manual: OPcache JIT settings](https://www.php.net/manual/en/opcache.configuration.php#ini.opcache.jit)
- [PHP.Watch: PHP 8.0 JIT](https://php.watch/versions/8.0/JIT)

Память, zval, ссылки, GC:

- [PHP Internals Book: zvals](https://www.phpinternalsbook.com/php7/zvals.html)
- [PHP Internals Book: Memory management](https://www.phpinternalsbook.com/php7/memory_management.html)
- [php-src: `Zend/zend_types.h`](https://github.com/php/php-src/blob/master/Zend/zend_types.h)
- [PHP Manual: References Explained](https://www.php.net/manual/en/language.references.php)
- [PHP Manual: What References Are Not](https://www.php.net/manual/en/language.references.arent.php)
- [PHP Manual: Garbage Collection](https://www.php.net/manual/en/features.gc.php)
- [PHP Manual: Collecting Cycles](https://www.php.net/manual/en/features.gc.collecting-cycles.php)
- [PHP Manual: GC functions](https://www.php.net/manual/en/ref.gc.php)
- [PHP Manual: `memory_get_usage`](https://www.php.net/manual/en/function.memory-get-usage.php)
- [PHP Manual: `memory_get_peak_usage`](https://www.php.net/manual/en/function.memory-get-peak-usage.php)
- [Nikita Popov: Internal value representation in PHP 7, part 1](https://www.npopov.com/2015/05/05/Internal-value-representation-in-PHP-7-part-1.html)
- [Nikita Popov: Internal value representation in PHP 7, part 2](https://www.npopov.com/2015/06/19/Internal-value-representation-in-PHP-7-part-2.html)
- [Nikita Popov: PHP internals, when does foreach copy?](https://www.npopov.com/2011/11/11/PHP-Internals-When-does-foreach-copy.html)

Composer:

- [Composer Docs: Basic usage](https://getcomposer.org/doc/01-basic-usage.md)
- [Composer Docs: CLI commands](https://getcomposer.org/doc/03-cli.md)
- [Composer Docs: composer.json schema](https://getcomposer.org/doc/04-schema.md)
- [Composer Docs: Versions and constraints](https://getcomposer.org/doc/articles/versions.md)
- [Composer Docs: Autoload](https://getcomposer.org/doc/04-schema.md#autoload)
- [Composer Docs: Autoloader optimization](https://getcomposer.org/doc/articles/autoloader-optimization.md)
- [Composer Docs: Config](https://getcomposer.org/doc/06-config.md)
- [Composer Docs: Platform config](https://getcomposer.org/doc/06-config.md#platform)
- [Composer Docs: Repositories](https://getcomposer.org/doc/05-repositories.md)
- [Composer Docs: Authentication for private packages](https://getcomposer.org/doc/articles/authentication-for-private-packages.md)
- [Composer Docs: Scripts](https://getcomposer.org/doc/articles/scripts.md)
- [Composer Docs: Plugins](https://getcomposer.org/doc/articles/plugins.md)
- [Composer Docs: allow-plugins](https://getcomposer.org/doc/06-config.md#allow-plugins)
- [Composer Docs: Audit](https://getcomposer.org/doc/03-cli.md#audit)

## PHP Request Lifecycle

Типичный HTTP-запрос через Nginx + PHP-FPM:

1. Клиент отправляет HTTP-запрос в Nginx.
2. Nginx выбирает location и передает PHP-запрос в PHP-FPM через FastCGI по Unix socket или TCP.
3. PHP-FPM master выбирает свободный worker process.
4. Worker принимает FastCGI-запрос и инициализирует request context.
5. PHP runtime запускает скрипт: bootstrap, Composer autoload, configs, DI/container, routing, controller.
6. Код выполняет бизнес-логику, I/O, запросы в БД, Redis, HTTP API.
7. Runtime формирует response body и headers.
8. Worker завершает request: shutdown-функции, request-bound resources, request memory arena cleanup.
9. Ответ возвращается в Nginx, затем клиенту.
10. Worker остается жить и ждет следующий запрос, если не достигнут `pm.max_requests` или аварийное завершение.

Ключевая мысль: PHP в FPM обычно не стартует процесс на каждый запрос. Процессы живут долго, но состояние приложения между HTTP-запросами должно считаться ненадежным и не использоваться как request state.

## Shared-Nothing Model

Shared-nothing означает, что каждый HTTP-запрос изолирован на уровне прикладного состояния:

- globals, static-переменные и объекты создаются заново для запроса;
- память, выделенная в рамках запроса, обычно освобождается после завершения;
- нет встроенного общего heap между запросами;
- для общего состояния используют БД, Redis, Memcached, файловое хранилище, очередь.

Что сохраняется между запросами:

- сам процесс FPM worker;
- загруженные расширения;
- OPcache shared memory;
- persistent connections, если включены и поддерживаются;
- process-level ресурсы;
- preloaded-классы и функции;
- static state может быть опасен в long-running workers, но обычный FPM-request должен проектироваться как stateless.

Common pitfalls:

- Думать, что PHP всегда запускается с нуля на каждый запрос.
- Хранить состояние пользователя в static/global вместо сессии, токена, Redis или БД.
- Не учитывать persistent connections и состояние транзакций/соединений.
- Переносить mental model из FPM в Swoole/RoadRunner без учета long-running процесса.

## SAPIs

SAPI, Server API, это слой интеграции PHP runtime с окружением выполнения.

Основные SAPI:

- `cli`: консольные команды, воркеры очередей, cron, Composer.
- `fpm-fcgi`: PHP-FPM через FastCGI, основной production-вариант.
- `apache2handler`: mod_php внутри Apache process.
- `phpdbg`: отладка и покрытие кода.
- `embed`: встраивание PHP в другое приложение.

Senior-answer: SAPI определяет, как PHP получает входные данные, отдает output, управляет lifecycle и какие ini-настройки применяются. CLI и FPM могут иметь разные `php.ini`, extensions, limits и OPcache settings.

Проверочный чеклист:

- Проверить `php -i | grep "Loaded Configuration File"` для CLI.
- Проверить `phpinfo()` или `php-fpm -i` для FPM.
- Сравнить `memory_limit`, `max_execution_time`, extensions, OPcache settings.
- Не дебажить FPM-проблему только через CLI-конфигурацию.

## PHP-FPM Architecture

PHP-FPM состоит из master process и worker processes.

Master process:

- читает конфиги;
- открывает listening socket;
- управляет pool-ами;
- создает и убивает воркеры;
- обрабатывает reload/stop signals;
- собирает базовый статус.

Worker process:

- принимает FastCGI-запрос;
- выполняет PHP-код;
- отдает ответ;
- после завершения запроса возвращается в пул.

Pool - отдельная группа воркеров со своими настройками: пользователь ОС, socket, лимиты, env, pm mode, slowlog. В одном FPM можно держать несколько pool-ов для разных приложений или классов нагрузки.

### Process Manager Modes

`pm = static`:

- всегда держит ровно `pm.max_children` процессов;
- предсказуемая latency и быстрый старт обработки;
- требует точного расчета памяти;
- часто подходит для стабильной production-нагрузки.

`pm = dynamic`:

- держит от `pm.min_spare_servers` до `pm.max_children`;
- стартует с `pm.start_servers`;
- баланс между памятью и готовностью к пикам;
- самый распространенный режим.

`pm = ondemand`:

- воркеры создаются по запросу и завершаются после `pm.process_idle_timeout`;
- экономит память на редкой нагрузке;
- может давать latency spike на создание процессов;
- полезен для dev/staging/редко используемых сервисов.

### `pm.max_children`

Формула:

```text
pm.max_children = доступная_память_для_FPM / средняя_или_p95_память_worker
```

Важно считать по реальному RSS/PSS под production-like нагрузкой, а не по `memory_limit`. `memory_limit` ограничивает heap PHP-запроса, но процесс может потреблять больше из-за extensions, OPcache, native allocations и фрагментации.

Пример:

```text
Сервер: 4 GB RAM
Оставить ОС, Nginx, MySQL sidecar, cache: 1 GB
FPM budget: 3 GB
p95 RSS worker: 120 MB
pm.max_children ~= 3000 / 120 = 25
```

### Production-чеклист FPM

- `pm.max_children` рассчитан по памяти и concurrency.
- `pm.max_requests` задан, чтобы периодически перезапускать воркеры при утечках/фрагментации.
- `request_terminate_timeout` защищает от бесконечных запросов.
- `slowlog` включен для диагностики долгих запросов.
- `pm.status_path` доступен только внутренне.
- Nginx `fastcgi_read_timeout` согласован с FPM timeout.
- `catch_workers_output` включают осознанно.
- Healthcheck не конкурирует с тяжелыми запросами.

### Timeouts и slowlog

Важные таймауты:

- `max_execution_time`: PHP-level лимит выполнения скрипта.
- `request_terminate_timeout`: FPM принудительно убивает worker, если запрос выполняется слишком долго.
- `fastcgi_read_timeout`: Nginx ждет ответ от FastCGI backend.
- DB/Redis/HTTP client timeout: должны быть заданы явно.

Практическое правило: timeout внешнего клиента должен быть меньше общего request budget. Иначе FPM worker будет висеть на I/O, а очередь запросов начнет расти.

Slowlog показывает stack trace PHP-кода для запросов, которые выполняются дольше `request_slowlog_timeout`.

```ini
request_slowlog_timeout = 3s
slowlog = /var/log/php-fpm/www-slow.log
```

Что искать:

- медленные SQL-запросы;
- сетевые вызовы без timeout;
- блокировки файлов/сессий;
- синхронные API-вызовы;
- тяжелый bootstrap;
- неэффективный autoload.

Pitfalls PHP-FPM:

- Поднять `pm.max_children` без расчета памяти и получить OOM killer.
- Ставить `memory_limit = -1` в web.
- Не задавать timeout для HTTP-клиентов и БД.
- Игнорировать session locking.
- Считать CPU-bound PHP-код масштабируемым только увеличением воркеров.

## OPcache

Без OPcache PHP должен часто читать, парсить и компилировать PHP-файлы заново. OPcache хранит скомпилированные opcodes в shared memory и переиспользует их между запросами и воркерами.

Что дает OPcache:

- меньше filesystem I/O;
- меньше CPU на parse/compile;
- быстрее bootstrap;
- стабильнее latency;
- база для preloading.

Ключевые настройки:

```ini
opcache.enable=1
opcache.enable_cli=0
opcache.memory_consumption=256
opcache.interned_strings_buffer=16
opcache.max_accelerated_files=20000
opcache.validate_timestamps=0
opcache.revalidate_freq=0
```

Для production часто ставят `validate_timestamps=0`, а сброс OPcache делают при deploy через reload FPM или explicit reset. Для dev обычно `validate_timestamps=1`.

OPcache pitfalls:

- `opcache.max_accelerated_files` меньше количества PHP-файлов проекта.
- Мало `opcache.memory_consumption`, частые restarts/cache full.
- В production включен timestamp validation с частой проверкой на большом проекте.
- Deploy заменяет файлы неатомарно, и OPcache видит смешанное состояние версий.
- CLI-команды тестируют без OPcache, а production работает с OPcache.

Диагностика:

- Посмотреть `opcache_get_status(false)`.
- Проверить `cache_full`, `oom_restarts`, `hash_restarts`, hit rate.
- Сравнить число cached scripts с числом файлов проекта.
- Проверить deploy strategy и FPM reload.
- Отдельно проверить `opcache.enable_cli` для CLI workers, если это нужно.

## Preloading

Preloading загружает выбранные PHP-файлы при старте FPM master до обработки запросов. Классы и функции становятся доступными воркерам без обычной загрузки файлов.

```ini
opcache.preload=/var/www/app/config/preload.php
opcache.preload_user=www-data
```

Плюсы:

- меньше runtime autoload/compile overhead;
- общие immutable структуры могут эффективнее использовать память;
- может ускорить framework bootstrap.

Минусы и риски:

- чтобы обновить preloaded-код, нужен restart/reload FPM master;
- нельзя рассчитывать на request-specific состояние;
- ошибки в preload ломают старт сервиса;
- mutable static state в preloaded-коде опасен;
- выгода зависит от приложения и уже настроенного OPcache.

Senior-answer: preloading полезен для стабильного набора классов фреймворка/домена, но требует дисциплины deploy и не заменяет нормальный OPcache/autoload optimization.

## JIT

JIT, Just-In-Time compilation, компилирует части opcode в машинный код во время выполнения. В PHP JIT завязан на OPcache.

```ini
opcache.jit_buffer_size=128M
opcache.jit=tracing
```

Когда JIT помогает:

- CPU-bound численные вычисления;
- tight loops;
- парсинг/обработка больших массивов без частого I/O;
- image/math/algorithmic workloads, если они написаны на PHP;
- некоторые CLI-задачи.

Когда JIT почти не помогает:

- типичный web CRUD;
- приложения, где bottleneck в БД, Redis, HTTP API, filesystem;
- framework-heavy request lifecycle с большим количеством dynamic dispatch;
- код, который большую часть времени проводит в C extensions;
- плохо профилированные приложения, где реальная проблема в I/O или N+1.

Практическая позиция: JIT не универсальный ускоритель web-приложений. Для большинства PHP backend-сервисов больший эффект дают OPcache, optimized autoload, SQL/index tuning, cache strategy, уменьшение I/O и правильный FPM sizing.

## Memory Model

PHP использует zval для представления значений. zval хранит тип и значение или указатель на структуру. Для массивов, строк и объектов используются отдельные структуры с reference counting.

Важные идеи:

- `memory_limit` ограничивает память PHP-скрипта, но не всегда всю память процесса.
- После запроса request memory обычно освобождается bulk-механизмами runtime.
- Массивы PHP гибкие, но тяжелые по памяти, потому что это ordered hash map.
- Объекты часто экономнее больших ассоциативных массивов, но зависят от структуры.
- Большие временные массивы могут резко повышать peak memory.

### zval

`zval` - базовый контейнер значения в Zend Engine. На уровне пользователя это не объект PHP, а внутренняя C-структура runtime, через которую PHP хранит значения переменных, элементов массива, свойств объекта, аргументов функций и временных результатов выражений.

Упрощенно `zval` содержит:

- `value`: само значение или указатель на более сложную структуру;
- `type info`: тип значения, например `IS_LONG`, `IS_DOUBLE`, `IS_STRING`, `IS_ARRAY`, `IS_OBJECT`, `IS_REFERENCE`, `IS_NULL`, `IS_FALSE`, `IS_TRUE`;
- служебные flags.

Не каждое значение хранится одинаково:

- `int`, `float`, `bool`, `null` обычно хранятся прямо в `zval` и не требуют reference counting;
- `string`, `array`, `object`, `resource`, `reference` указывают на refcounted-структуры;
- refcount показывает, сколько `zval` ссылаются на одну внутреннюю структуру;
- если refcounted-структуры образуют цикл, обычного refcount недостаточно, нужен GC cycle collector.

### Copy-on-Write

PHP value semantics выглядят как копирование при присваивании, но реализация оптимизирована через refcount + copy-on-write. Поэтому чтение и передача по значению часто дешевы, а запись в разделяемую структуру может быть дорогой.

```php
$a = [1, 2, 3];
$b = $a;      // refcount общей HashTable увеличен
$b[] = 4;     // PHP отделяет $b: создается копия массива
```

Практический вывод: передача больших массивов по значению обычно не страшна, пока нет модификации. Но неочевидная модификация внутри функции может привести к большому copy-on-write.

### References

References в PHP не являются C-поинтерами. Это alias на zval/reference container.

```php
$items = [1, 2, 3];

foreach ($items as &$item) {
    $item *= 2;
}

unset($item); // важно разорвать reference
```

Pitfalls:

- `foreach` by reference без `unset($item)` после цикла.
- Использование references для оптимизации без измерений.
- Неожиданные aliasing-баги при передаче по ссылке.
- References могут мешать copy-on-write оптимизациям.

Senior-answer: references нужны редко: для API, которые явно требуют mutation by reference, или для специфичных структур. В большинстве бизнес-кода они ухудшают читаемость.

### Arrays, Strings, Objects

Строки:

- immutable на уровне пользовательской модели;
- refcounted;
- присваивание строки дешево до изменения;
- конкатенация и изменение создают новые буферы или отделяют значение.

Массивы:

- реализованы как ordered hash table;
- refcounted;
- очень гибкие, но тяжелые по памяти;
- запись в разделяемый массив запускает separation;
- большие массивы с ассоциативными ключами могут потреблять намного больше памяти, чем ожидается.

Объекты:

- переменная хранит handle/указатель на объектную структуру;
- присваивание объекта не копирует объект, а копирует handle;
- изменение свойства через одну переменную видно через другую;
- для копии нужен явный `clone`.

Короткое объяснение: массивы и строки имеют value semantics с copy-on-write, объекты имеют object identity semantics.

## Garbage Collector

PHP в основном использует reference counting. Когда refcount падает до нуля, память освобождается. Но reference counting не может освободить циклические ссылки.

```php
final class Node
{
    public ?Node $next = null;
}

$a = new Node();
$b = new Node();
$a->next = $b;
$b->next = $a;

unset($a, $b); // refcount внутри цикла не ноль, нужен GC cycle collector
```

Полезные функции:

```php
gc_enabled();
gc_enable();
gc_disable();
gc_collect_cycles();
gc_status();
```

Когда думать о GC:

- большие графы объектов;
- циклические ссылки: parent-child, event listeners, closures capturing `$this`;
- long-running workers;
- batch processing;
- демоны очередей.

## Memory Leaks in Long-Running Workers

В FPM утечки часто маскируются завершением запроса и `pm.max_requests`. В long-running CLI workers, RoadRunner, Swoole, ReactPHP, Laravel Octane утечки накапливаются.

Частые причины:

- static caches без ограничения размера;
- singleton services с request-specific данными;
- event listeners/subscribers не отписываются;
- closures удерживают большие объекты;
- Doctrine EntityManager не очищается после job/batch;
- Monolog buffers/handlers копят записи;
- Guzzle/HTTP clients держат историю или middleware state;
- большие массивы не освобождаются между итерациями;
- native extensions leaking memory.

Практический чеклист worker-а:

- Ограничить `--max-jobs`, `--max-time`, memory limit supervisor-а.
- После job очищать ORM UnitOfWork/EntityManager.
- Сбрасывать request/job scoped services.
- Не хранить DTO/Entity в static/global cache.
- Использовать bounded cache.
- Логировать `memory_get_usage(true)` и `memory_get_peak_usage(true)`.
- Периодически `gc_collect_cycles()` после больших batch-операций, если есть циклы.

## Composer Dependency Resolution

Composer решает задачу SAT-solving: подобрать версии пакетов, которые удовлетворяют всем constraints из `composer.json`, зависимостей зависимостей, platform requirements и stability rules.

Ключевые команды:

```bash
composer install
composer update
composer require vendor/package:^1.2
composer why vendor/package
composer why-not vendor/package 2.0
composer outdated
composer audit
```

`composer install`:

- читает `composer.lock`;
- ставит ровно зафиксированные версии;
- должен использоваться в CI/production deploy.

`composer update`:

- пересчитывает dependency graph;
- обновляет `composer.lock`;
- должен выполняться осознанно и обычно локально/в отдельном PR.

`composer.lock` фиксирует точные версии пакетов, source/dist refs, platform и metadata. Для приложений `composer.lock` коммитят, чтобы все окружения получали одинаковые зависимости. Для библиотек lock-файл обычно не нужен потребителям, но может использоваться для CI самой библиотеки.

Pitfalls:

- Запускать `composer update` на production.
- Не коммитить lock в приложении.
- Обновлять весь dependency graph ради одного пакета без необходимости.
- Игнорировать `composer why-not` при конфликте версий.

## Semantic Versioning и Constraints

SemVer: `MAJOR.MINOR.PATCH`.

- MAJOR: breaking changes.
- MINOR: новая совместимая функциональность.
- PATCH: bug fixes.

Популярные constraints:

```json
{
  "require": {
    "vendor/a": "^2.3",
    "vendor/b": "~1.4",
    "vendor/c": "1.2.*",
    "vendor/d": ">=1.0 <2.0"
  }
}
```

Смысл:

- `^2.3`: `>=2.3.0 <3.0.0`.
- `^0.3`: `>=0.3.0 <0.4.0`, потому что `0.x` считается нестабильным API.
- `~1.4`: обычно `>=1.4.0 <2.0.0`.
- `1.2.*`: любые patch в `1.2`.

Практическая позиция: для приложений обычно используют caret constraints и lock-файл. Для критичных пакетов можно сужать диапазон, но слишком жесткие constraints ухудшают обновляемость.

## Composer Autoload

PSR-4 мапит namespace prefix на директорию.

```json
{
  "autoload": {
    "psr-4": {
      "App\\": "src/"
    }
  }
}
```

Classmap содержит явную карту `class => file` и полезен для legacy-кода без PSR-4.

Production-команда:

```bash
composer install --no-dev --prefer-dist --optimize-autoloader --classmap-authoritative
```

Что делает:

- `--optimize-autoloader` генерирует classmap для PSR-4/PSR-0;
- `--classmap-authoritative` запрещает fallback filesystem lookup, если класса нет в classmap;
- `--apcu-autoloader` может кешировать autoload lookups в APCu.

Pitfalls:

- Использовать `classmap-authoritative`, когда приложение генерирует классы после install.
- Забыть `composer dump-autoload` после изменения autoload mapping.
- Смешивать неправильный namespace и path case, что ломается на Linux.
- Держать dev dependencies в production.

## Platform Config

Composer учитывает platform packages: PHP version, extensions, libs.

```json
{
  "config": {
    "platform": {
      "php": "8.3.0",
      "ext-redis": "6.0.0"
    }
  }
}
```

Зачем:

- резолвить зависимости под production PHP, даже если локально другая версия;
- предотвращать случайную установку пакета, несовместимого с целевой платформой;
- стабилизировать CI/deploy.

Pitfalls:

- `config.platform.php` не меняет реальную версию PHP runtime.
- Можно скрыть отсутствие extension, если неправильно зафиксировать platform.
- Нельзя игнорировать `composer check-platform-reqs` перед deploy.

## Private Packages, Scripts, Plugins Security

Варианты private packages:

- Private VCS repository через `repositories`.
- Private Packagist.
- Satis.
- Artifact repository.
- GitHub/GitLab package registry.

Security checklist:

- Не коммитить токены в `composer.json`, `auth.json`, `.env`.
- Использовать CI secrets.
- Ограничивать scope токенов.
- Предпочитать deploy keys или read-only tokens.
- Проверять supply chain: vendor ownership, abandoned packages, audit.

Composer scripts могут выполнять команды на событиях `post-install-cmd`, `post-update-cmd`, `post-autoload-dump`. Composer plugins могут расширять поведение Composer и выполнять код во время install/update.

Риски:

- выполнение произвольного кода из dependency graph;
- compromised package может добавить вредоносный script/plugin;
- CI secrets могут утечь через scripts;
- plugin может менять install behavior.

Практические меры:

```bash
composer install --no-scripts
composer install --no-plugins
composer audit
```

Senior-answer: в modern Composer plugins должны быть явно разрешены через `allow-plugins`. В CI важно понимать, какие scripts/plugins выполняются, и не запускать install/update из недоверенного PR с доступом к секретам.

## Финальная формулировка

PHP в production чаще всего работает как пул долгоживущих FPM-процессов с изолированным request lifecycle. Производительность держится на правильном FPM sizing, контроле I/O и timeout-ов, OPcache, разумном autoload-е и воспроизводимых зависимостях. JIT и preloading полезны только после профилирования и дисциплинированного deploy. Память в PHP удобна благодаря request cleanup и copy-on-write, но references, циклы и long-running workers требуют аккуратности. Composer надо воспринимать не как простой downloader, а как dependency solver и часть supply-chain security.
