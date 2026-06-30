# За 1 день: PHP runtime, FPM, OPcache, JIT, memory, GC, Composer

Цель: быстро освежить системное понимание PHP runtime для собеседования Lead/Senior PHP developer. Фокус на практических объяснениях, типичных ловушках, диагностике и коротких senior-level ответах.

## Как заниматься 1 день

1. 60 минут: request lifecycle, shared-nothing, SAPI.
2. 90 минут: PHP-FPM, воркеры, таймауты, slowlog, тюнинг.
3. 90 минут: OPcache, preloading, JIT.
4. 90 минут: память, copy-on-write, references, GC, long-running workers.
5. 90 минут: Composer, autoload, lock-файл, security.
6. 60 минут: самопроверка и проговаривание коротких ответов.

## PHP Request Lifecycle

Типичный HTTP-запрос через Nginx + PHP-FPM:

1. Клиент отправляет HTTP-запрос в Nginx.
2. Nginx выбирает location и передает PHP-запрос в PHP-FPM через FastCGI по Unix socket или TCP.
3. PHP-FPM master выбирает свободный worker process.
4. Worker принимает FastCGI-запрос, инициализирует request context.
5. PHP runtime запускает скрипт: загружает bootstrap, автолоадер, конфиги, DI/container, роутинг, контроллер.
6. Код выполняет бизнес-логику, I/O, запросы в БД, Redis, HTTP API.
7. Runtime формирует response body и headers.
8. Worker завершает request: вызывает shutdown-функции, закрывает request-bound ресурсы, освобождает request memory arena.
9. Ответ возвращается в Nginx, затем клиенту.
10. Worker остается жить и ждет следующий запрос, если не достигнут `pm.max_requests` или аварийное завершение.

Ключевая мысль: PHP в FPM обычно не стартует процесс на каждый запрос. Процессы живут долго, но состояние приложения между HTTP-запросами должно считаться ненадежным и не использоваться как request state.

### Shared-Nothing Model

Shared-nothing означает, что каждый HTTP-запрос изолирован на уровне прикладного состояния:

- глобальные переменные, static-переменные и объекты создаются заново для запроса;
- память, выделенная в рамках запроса, обычно освобождается после завершения запроса;
- нет встроенного общего heap между запросами;
- для общего состояния используют внешние хранилища: БД, Redis, Memcached, файловое хранилище, очередь.

Практическое объяснение на интервью: PHP-FPM worker переиспользуется, но request context пересоздается. Это упрощает модель отказов и утечек по сравнению с постоянно живущим приложением, но не отменяет проблему утечек в расширениях, статических кешах, preloaded-коде и long-running CLI workers.

### Что сохраняется между запросами

- сам процесс FPM worker;
- загруженные расширения;
- OPcache shared memory;
- persistent connections, если включены и поддерживаются;
- некоторые process-level ресурсы;
- preloaded-классы и функции в master/worker process memory;
- static state в редких сценариях может вести себя неожиданно при long-running workers, но обычный FPM-request должен проектироваться как stateless.

### Common Pitfalls

- Думать, что PHP всегда запускается с нуля на каждый запрос.
- Хранить состояние пользователя в static/global вместо сессии, токена, Redis или БД.
- Не учитывать, что persistent connections могут переживать запрос и создавать проблемы с транзакциями/состоянием соединения.
- Переносить mental model из FPM в Swoole/RoadRunner без учета long-running процесса.

## SAPIs

SAPI, Server API, это слой интеграции PHP runtime с окружением выполнения.

Основные SAPI:

- `cli`: консольные команды, воркеры очередей, cron, Composer.
- `fpm-fcgi`: PHP-FPM через FastCGI, основной production-вариант для Nginx/Apache.
- `apache2handler`: mod_php внутри Apache process.
- `phpdbg`: отладка и покрытие кода.
- `embed`: встраивание PHP в другое приложение.

Короткий senior-answer: SAPI определяет, как PHP получает входные данные, отдает output, управляет request lifecycle и какие ini-настройки применяются. CLI и FPM могут иметь разные `php.ini`, extensions и memory/time limits.

### Проверочный чеклист

- Проверить `php -i | grep "Loaded Configuration File"` для CLI.
- Проверить `phpinfo()` или `php-fpm -i` для FPM.
- Сравнить `memory_limit`, `max_execution_time`, extensions, OPcache settings.
- Не дебажить FPM-проблему только через CLI-конфигурацию.

## PHP-FPM Architecture

Официальная база по теме: [PHP-FPM в manual PHP](https://www.php.net/manual/en/install.fpm.php) и [настройки PHP-FPM](https://www.php.net/manual/en/install.fpm.configuration.php).

PHP-FPM состоит из master process и worker processes.

Master process:

- читает конфиги;
- открывает listening socket;
- управляет pool-ами;
- создает и убивает воркеры;
- обрабатывает сигналы reload/stop;
- собирает базовый статус.

Worker process:

- принимает FastCGI-запрос;
- выполняет PHP-код;
- отдает ответ;
- после завершения запроса возвращается в пул.

Pool, или пул, это отдельная группа воркеров со своими настройками: пользователь ОС, socket, лимиты, env, pm mode, slowlog. В одном FPM можно держать несколько pool-ов для разных приложений или классов нагрузки.

### Process Manager Modes

`pm = static`

- Всегда держит ровно `pm.max_children` процессов.
- Предсказуемая latency, быстрый старт обработки.
- Требует точного расчета памяти.
- Часто подходит для стабильной production-нагрузки.

`pm = dynamic`

- Держит от `pm.min_spare_servers` до `pm.max_children`.
- Стартует с `pm.start_servers`.
- Баланс между памятью и готовностью к пикам.
- Самый распространенный режим.

`pm = ondemand`

- Воркеры создаются по запросу и завершаются после `pm.process_idle_timeout`.
- Экономит память на редкой нагрузке.
- Может давать latency spike на создание процессов.
- Полезен для dev/staging/редко используемых сервисов.

### Базовый расчет `pm.max_children`

Формула:

```text
pm.max_children = доступная_память_для_FPM / средняя_или_p95_память_worker
```

Пример:

```text
Сервер: 4 GB RAM
Оставить ОС, Nginx, MySQL sidecar, cache: 1 GB
FPM budget: 3 GB
p95 RSS worker: 120 MB
pm.max_children ~= 3000 / 120 = 25
```

Важно: считать по реальному RSS/PSS под production-like нагрузкой, а не по `memory_limit`. `memory_limit` ограничивает heap PHP-запроса, но процесс может потреблять больше из-за extensions, OPcache, native allocations и фрагментации.

### Tuning Basics

Минимальный production-чеклист:

- `pm.max_children` рассчитан по памяти и concurrency.
- `pm.max_requests` задан, чтобы периодически перезапускать воркеры при утечках/фрагментации.
- `request_terminate_timeout` защищает от бесконечных запросов.
- `slowlog` включен для диагностики долгих запросов.
- `pm.status_path` доступен только внутренне.
- Nginx `fastcgi_read_timeout` согласован с FPM timeout.
- `catch_workers_output` включают осознанно, чтобы не потерять stderr/stdout, но не устроить шум в логах.
- Healthcheck не конкурирует с тяжелыми запросами.

### Timeouts

Важные таймауты:

- `max_execution_time`: PHP-level лимит выполнения скрипта, чаще для web SAPI, не всегда учитывает blocking I/O одинаково во всех сценариях.
- `request_terminate_timeout`: FPM принудительно убивает worker, если запрос выполняется слишком долго.
- `fastcgi_read_timeout`: Nginx ждет ответ от FastCGI backend.
- DB/Redis/HTTP client timeout: должны быть заданы явно.

Практическое правило: timeout внешнего клиента должен быть меньше общего request budget. Иначе FPM worker будет висеть на I/O, а очередь запросов начнет расти.

### Slowlog

Slowlog показывает stack trace PHP-кода для запросов, которые выполняются дольше `request_slowlog_timeout`.

Пример настроек:

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

### Pitfalls PHP-FPM

- Поднять `pm.max_children` без расчета памяти и получить OOM killer.
- Ставить `memory_limit = -1` в web.
- Не задавать timeout для HTTP-клиентов и БД.
- Игнорировать session locking: параллельные запросы одного пользователя могут ждать `session_start()`/закрытия сессии.
- Считать CPU-bound PHP-код масштабируемым только увеличением воркеров. Если CPU уже насыщен, больше воркеров ухудшит latency.

## OPcache

Официальные страницы: [OPcache](https://www.php.net/manual/en/book.opcache.php) и [OPcache runtime configuration](https://www.php.net/manual/en/opcache.configuration.php).

Без OPcache PHP должен читать, парсить и компилировать PHP-файлы в opcode часто заново. OPcache хранит скомпилированные opcodes в shared memory и переиспользует их между запросами и воркерами.

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

### OPcache Pitfalls

- `opcache.max_accelerated_files` меньше количества PHP-файлов проекта.
- Мало `opcache.memory_consumption`, частые restarts/cache full.
- В production включен timestamp validation с частой проверкой на очень большом проекте.
- Deploy заменяет файлы неатомарно, и OPcache видит смешанное состояние версий.
- CLI-команды тестируют без OPcache, а production работает с OPcache.

### Мини-чеклист диагностики OPcache

- Посмотреть `opcache_get_status(false)`.
- Проверить `cache_full`, `oom_restarts`, `hash_restarts`, hit rate.
- Сравнить число cached scripts с числом файлов проекта.
- Проверить deploy strategy и FPM reload.
- Отдельно проверить `opcache.enable_cli` для CLI workers, если это нужно.

## Preloading

Официальная документация: [OPcache preloading](https://www.php.net/manual/en/opcache.preloading.php).

Preloading загружает выбранные PHP-файлы при старте FPM master до обработки запросов. Классы и функции становятся доступными воркерам без обычной загрузки файлов.

Пример:

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

Официальная документация: [OPcache JIT](https://www.php.net/manual/en/opcache.configuration.php#ini.opcache.jit) и [JIT configuration](https://www.php.net/manual/en/opcache.configuration.php#ini.opcache.jit-buffer-size).

JIT, Just-In-Time compilation, компилирует части opcode в машинный код во время выполнения. В PHP JIT завязан на OPcache.

Пример настроек:

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

Практическая позиция: JIT не является универсальным ускорителем web-приложений. Для большинства PHP backend-сервисов больший эффект дают OPcache, optimized autoload, SQL/index tuning, cache strategy, уменьшение I/O и правильный FPM sizing.

### JIT Pitfalls

- Включить JIT в production без профилирования и ожидать кратного ускорения.
- Забыть выделить `opcache.jit_buffer_size`.
- Сравнивать microbenchmark с реальным web latency.
- Не учесть рост memory footprint и сложность диагностики.

## Memory Model

Для углубления: [официальная документация по ссылкам в PHP](https://www.php.net/manual/en/language.references.php), [сборщик циклических ссылок](https://www.php.net/manual/en/features.gc.collecting-cycles.php) и [управление GC](https://www.php.net/manual/en/ref.gc.php).

PHP использует zval для представления значений. zval хранит тип и значение или указатель на структуру. Для массивов, строк и объектов используются отдельные структуры с reference counting.

Важные идеи:

- `memory_limit` ограничивает память PHP-скрипта, но не всегда всю память процесса.
- После запроса request memory обычно освобождается bulk-механизмами runtime.
- Массивы PHP гибкие, но тяжелые по памяти, потому что это ordered hash map.
- Объекты часто экономнее больших ассоциативных массивов, но зависят от структуры.
- Большие временные массивы могут резко повышать peak memory.

### zval: Что Это Такое

`zval` — базовый контейнер значения в Zend Engine. На уровне пользователя это не объект PHP, а внутренняя C-структура runtime, через которую PHP хранит значения переменных, элементов массива, свойств объекта, аргументов функций и временных результатов выражений.

Упрощенно `zval` содержит:

- `value`: само значение или указатель на более сложную структуру;
- `type info`: тип значения, например `IS_LONG`, `IS_DOUBLE`, `IS_STRING`, `IS_ARRAY`, `IS_OBJECT`, `IS_REFERENCE`, `IS_NULL`, `IS_FALSE`, `IS_TRUE`;
- служебные flags, которые помогают runtime понять, как значение хранить, копировать и освобождать.

Важно: это высокоуровневая модель для интервью. Детали зависят от версии PHP и реализации Zend Engine. Для точной структуры смотри [PHP Internals Book: zvals](https://www.phpinternalsbook.com/php7/zvals.html) и `zend_types.h` в [php-src](https://github.com/php/php-src/blob/master/Zend/zend_types.h).

### zval и refcounted values

Не каждое значение хранится одинаково:

- `int`, `float`, `bool`, `null` обычно хранятся прямо в `zval` и не требуют reference counting;
- `string`, `array`, `object`, `resource`, `reference` указывают на отдельные refcounted-структуры;
- refcount показывает, сколько `zval` ссылаются на одну внутреннюю структуру;
- когда refcount падает до нуля, структура может быть освобождена;
- если refcounted-структуры образуют цикл, обычного refcount недостаточно, нужен GC cycle collector.

Пример mental model:

```php
$a = 'hello';
$b = $a;
```

После присваивания два `zval` могут ссылаться на одну refcounted-строку. Реального копирования строки нет, пока не понадобится изменить одно из значений.

### zval, `is_ref` и references

В старых объяснениях часто встречается термин `is_ref`. В современных PHP полезнее думать не как о простом флаге "это ссылка", а как о специальном контейнере `IS_REFERENCE`, который оборачивает значение и позволяет нескольким переменным быть aliases одного контейнера.

Пример:

```php
$a = 1;
$b =& $a;
$b = 2;

// $a === 2, потому что $a и $b указывают на один reference container
```

Практическая суть:

- обычное присваивание создает еще один `zval`, который может разделять refcounted value до изменения;
- присваивание по ссылке `=&` связывает переменные как aliases;
- `IS_REFERENCE` влияет на copy-on-write и может делать поведение менее очевидным;
- references почти никогда не нужны для экономии памяти в бизнес-коде.

### zval и Copy-on-Write

Copy-on-write строится вокруг `zval` и refcounted-структур. Если массив или строка разделяются несколькими `zval`, PHP откладывает копирование до первой записи.

```php
$a = [1, 2, 3];
$b = $a;      // refcount общей HashTable увеличен
$b[] = 4;     // PHP отделяет $b: создается копия массива
```

Senior-level формулировка: PHP value semantics выглядят как "копирование при присваивании", но реализация оптимизирована через refcount + copy-on-write. Поэтому чтение и передача по значению часто дешевы, а запись в разделяемую структуру может быть дорогой.

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
- изменение свойства через одну переменную видно через другую, потому что это тот же объект;
- для копии нужен явный `clone`.

Пример:

```php
$a = new stdClass();
$b = $a;
$b->name = 'PHP';

// $a->name тоже 'PHP'
```

Короткое объяснение: массивы и строки имеют value semantics с copy-on-write, объекты имеют object identity semantics. Это частая причина ошибок при переносе ожиданий с массивов на объекты и обратно.

### zval, память и GC

Связь с памятью:

- каждый элемент массива — не только пользовательское значение, но и служебные структуры HashTable/bucket/zval;
- `array<int, mixed>` в PHP значительно тяжелее плотного массива в C/Java;
- временные массивы в `array_map`, `array_filter`, `array_merge` могут кратно поднять peak memory;
- references могут мешать оптимизациям и удерживать значения дольше ожидаемого;
- объекты с циклическими ссылками требуют участия GC.

Связь с GC:

- refcount освобождает большинство значений сразу;
- GC нужен для циклов из refcounted values;
- `zval` сам по себе не означает утечку, но граф refcounted-структур может стать недостижимым для кода и при этом иметь ненулевые refcount внутри цикла;
- в long-running worker-ах важно очищать графы объектов и bounded caches, иначе память растет между job-ами.

Мини-чеклист для практики:

- Если peak memory высокий, ищи большие промежуточные массивы.
- Если memory растет в worker-е, ищи static cache, service singleton state, event listeners, ORM UnitOfWork и циклы объектов.
- Если хочешь "оптимизировать ссылками", сначала измерь: часто станет хуже по читаемости и не лучше по памяти.
- Если массив огромный и однотипный, подумай о streaming, generators, `SplFixedArray`, DTO/objects, chunk processing или переносе вычислений в БД/extension.

### Вопросы про zval и короткие ответы

Вопрос: Что такое `zval`?


Ответ: Внутренний контейнер значения в Zend Engine. Он хранит тип и значение или указатель на refcounted-структуру вроде строки, массива, объекта или reference container.

Вопрос: PHP копирует массив при `$b = $a`?


Ответ: Не сразу. Обычно увеличивается refcount общей HashTable. Копия появляется при записи в одну из переменных, если структура разделяется.

Вопрос: Почему объекты ведут себя не как массивы?


Ответ: Переменная с объектом хранит handle на объект. Присваивание копирует handle, а не объект. Поэтому изменение свойства видно через все переменные, указывающие на тот же объект.

Вопрос: Что такое `IS_REFERENCE`?


Ответ: Внутренний reference container, который позволяет нескольким переменным быть aliases одного значения. Он появляется при использовании PHP references, например `=&` или `foreach` by reference.

Вопрос: Как `zval` связан с GC?


Ответ: Многие `zval` указывают на refcounted-структуры. Refcount освобождает обычные значения, но циклы refcounted-объектов/массивов требует cycle collector.

Вопрос: Как объяснить memory overhead массивов PHP?


Ответ: PHP array — ordered hash map, а не компактный C-array. Каждый элемент несет bucket, ключ, zval и служебные данные, поэтому большие массивы дорогие по памяти.

### Copy-on-Write

PHP копирует массивы/строки лениво. Присваивание не копирует данные сразу, а увеличивает refcount. Реальная копия происходит при изменении.

Пример:

```php
$a = range(1, 1000000);
$b = $a;        // дешево: общая структура, refcount++
$b[0] = 42;     // дорого: отделение копии массива
```

Практический вывод: передача больших массивов по значению обычно не страшна, пока нет модификации. Но неочевидная модификация внутри функции может привести к большому copy-on-write.

### References

References в PHP не являются C-поинтерами. Это alias на zval/container.

Опасный пример:

```php
$items = [1, 2, 3];

foreach ($items as &$item) {
    $item *= 2;
}

unset($item); // важно разорвать reference

foreach ($items as $item) {
    // безопасно после unset
}
```

Pitfalls:

- `foreach` by reference без `unset($item)` после цикла.
- Использование references для "оптимизации" без измерений.
- Неожиданные aliasing-баги при передаче по ссылке.
- References могут мешать copy-on-write оптимизациям.

Senior-answer: references нужны редко: для API, которые явно требуют mutation by reference, или для специфичных структур. В большинстве бизнес-кода они ухудшают читаемость и могут ломать ожидания copy-on-write.

## Garbage Collector

PHP в основном использует reference counting. Когда refcount падает до нуля, память освобождается. Но reference counting сам по себе не может освободить циклические ссылки.

Пример цикла:

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

Garbage collector ищет и собирает циклические структуры. Обычно он включен по умолчанию.

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

### Memory Leaks in Long-Running Workers

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

Мини-пример batch:

```php
foreach ($chunks as $chunk) {
    processChunk($chunk);

    unset($chunk);
    $entityManager->clear();
    gc_collect_cycles();
}
```

## Composer Dependency Resolution

Официальная база: [Composer basic usage](https://getcomposer.org/doc/01-basic-usage.md), [CLI commands](https://getcomposer.org/doc/03-cli.md), [composer.json schema](https://getcomposer.org/doc/04-schema.md).

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

### composer.lock

`composer.lock` фиксирует точные версии пакетов, source/dist refs, platform и metadata.

Senior-answer: для приложений `composer.lock` коммитят, чтобы все окружения получали одинаковые зависимости. Для библиотек lock-файл обычно не нужен потребителям, но может использоваться для CI самой библиотеки.

Pitfalls:

- Запускать `composer update` на production.
- Не коммитить lock в приложении.
- Обновлять весь dependency graph ради одного пакета без необходимости.
- Игнорировать `composer why-not` при конфликте версий.

## Semantic Versioning и Constraints

Официальная справка Composer: [версии и constraints](https://getcomposer.org/doc/articles/versions.md).

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

Официальная справка Composer: [autoload в composer.json](https://getcomposer.org/doc/04-schema.md#autoload) и [оптимизация autoloader-а](https://getcomposer.org/doc/articles/autoloader-optimization.md).

### PSR-4

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

Класс `App\Service\UserService` будет искаться в `src/Service/UserService.php`.

### Classmap

Classmap содержит явную карту `class => file`.

```json
{
  "autoload": {
    "classmap": ["legacy/"]
  }
}
```

Полезно для legacy-кода без PSR-4.

### Optimized Autoload

Production-команда:

```bash
composer install --no-dev --prefer-dist --optimize-autoloader --classmap-authoritative
```

Что делает:

- `--optimize-autoloader` генерирует classmap для PSR-4/PSR-0;
- `--classmap-authoritative` запрещает fallback filesystem lookup, если класса нет в classmap;
- `--apcu-autoloader` может кешировать autoload lookups в APCu, если расширение доступно и сценарий подходит.

Pitfalls:

- Использовать `classmap-authoritative`, когда приложение генерирует классы после install.
- Забыть `composer dump-autoload` после изменения autoload mapping.
- Смешивать неправильный namespace и path case, что ломается на Linux.
- Держать dev dependencies в production.

## Platform Config

Официальная справка Composer: [`config.platform`](https://getcomposer.org/doc/06-config.md#platform) и [`check-platform-reqs`](https://getcomposer.org/doc/03-cli.md#check-platform-reqs).

Composer учитывает platform packages: PHP version, extensions, libs.

Пример:

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

Полезная команда:

```bash
composer check-platform-reqs
```

## Private Packages

Официальная справка Composer: [repositories](https://getcomposer.org/doc/05-repositories.md), [authentication](https://getcomposer.org/doc/articles/authentication-for-private-packages.md), [Private Packagist](https://packagist.com/).

Варианты:

- Private VCS repository через `repositories`.
- Private Packagist.
- Satis.
- Artifact repository.
- GitHub/GitLab package registry.

Пример VCS:

```json
{
  "repositories": [
    {
      "type": "vcs",
      "url": "git@github.com:company/private-package.git"
    }
  ],
  "require": {
    "company/private-package": "^1.0"
  }
}
```

Security checklist:

- Не коммитить токены в `composer.json`, `auth.json`, `.env`.
- Использовать CI secrets.
- Ограничивать scope токенов.
- Предпочитать deploy keys или read-only tokens.
- Проверять supply chain: vendor ownership, abandoned packages, audit.

## Scripts и Plugins Security

Официальная справка Composer: [scripts](https://getcomposer.org/doc/articles/scripts.md), [plugins](https://getcomposer.org/doc/articles/plugins.md), [`allow-plugins`](https://getcomposer.org/doc/06-config.md#allow-plugins), [`composer audit`](https://getcomposer.org/doc/03-cli.md#audit).

Composer scripts могут выполнять команды на событиях `post-install-cmd`, `post-update-cmd`, `post-autoload-dump` и других.

Composer plugins могут расширять поведение Composer и выполнять код во время install/update.

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

В `composer.json`:

```json
{
  "config": {
    "allow-plugins": {
      "composer/package-versions-deprecated": false,
      "phpstan/extension-installer": true
    }
  }
}
```

Senior-answer: в modern Composer plugins должны быть явно разрешены через `allow-plugins`. В CI важно понимать, какие scripts/plugins выполняются, и не запускать install/update из недоверенного PR с доступом к секретам.

## Практические Mini-Practice

### 1. Рассчитать FPM workers

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

Но если workload CPU-bound, 40 воркеров могут ухудшить latency. Нужно смотреть CPU saturation, run queue, response time, queue length и DB limits.

### 2. Найти проблему в FPM

Симптомы:

- Nginx иногда отдает 502/504.
- FPM log: `server reached pm.max_children`.
- DB CPU 95%.

Вероятное объяснение: не обязательно мало воркеров. Возможно, запросы висят на медленной БД, воркеры заняты ожиданием, очередь растет. Увеличение `pm.max_children` может добить БД. Нужно slowlog, DB slow query log, tracing, индексы, таймауты, backpressure.

### 3. Copy-on-write

Вопрос:

```php
function touchFirst(array $data): array
{
    $data[0] = 42;
    return $data;
}

$items = range(1, 1000000);
$new = touchFirst($items);
```

Что произойдет? При передаче в функцию копии сразу нет, но при `$data[0] = 42` произойдет separation/copy массива, потому что исходный `$items` еще ссылается на те же данные.

### 4. Composer conflict

Если `composer update vendor/package` не может поставить версию `2.0`, действия:

```bash
composer why-not vendor/package 2.0
composer why vendor/conflicting-package
composer outdated --direct
```

Дальше решить: обновить зависимый пакет, расширить constraint, заменить пакет, временно остаться на старой версии.

## Вопросы для самопроверки и короткие ответы

### PHP Runtime

Вопрос: PHP-FPM создает новый процесс на каждый запрос?


Ответ: Нет. FPM держит пул worker processes. Worker обрабатывает запрос, очищает request context и переиспользуется для следующих запросов.

Вопрос: Что означает shared-nothing?


Ответ: Прикладное состояние запроса не разделяется с другими запросами через общий heap. Для общего состояния нужны внешние системы: БД, Redis, очередь, storage.

Вопрос: Почему CLI и FPM могут вести себя по-разному?


Ответ: Это разные SAPI, у них могут быть разные `php.ini`, extensions, limits, OPcache settings и environment.

### PHP-FPM

Вопрос: Как выбрать `pm.max_children`?


Ответ: По доступной памяти и реальному потреблению worker-а под нагрузкой, затем проверить CPU, DB limits и latency. Формула: FPM memory budget / p95 RSS worker.

Вопрос: Зачем `pm.max_requests`?


Ответ: Чтобы периодически перезапускать воркеры и ограничивать накопление утечек, фрагментации памяти или проблем в extensions.

Вопрос: Что делать при `server reached pm.max_children`?


Ответ: Не просто увеличивать лимит. Проверить slowlog, DB/API latency, CPU, memory, queue, timeout-и. Причина может быть в долгих запросах, а не в малом количестве воркеров.

### OPcache и JIT

Вопрос: Что ускоряет OPcache?


Ответ: Повторное использование скомпилированных opcodes из shared memory вместо чтения, парсинга и компиляции PHP-файлов на каждом запросе.

Вопрос: Когда нужен `opcache.validate_timestamps=0`?


Ответ: В production, если deploy атомарный и OPcache сбрасывается/reload-ится при релизе. Это убирает регулярные проверки mtime файлов.

Вопрос: Почему JIT редко ускоряет обычный web CRUD?


Ответ: Bottleneck обычно в I/O, БД, сети, framework bootstrap и C extensions, а не в CPU-bound PHP loops.

### Memory и GC

Вопрос: Что такое copy-on-write?


Ответ: При присваивании массивов/строк PHP не копирует данные сразу. Копия создается только при изменении одного из владельцев.

Вопрос: Почему `foreach` по ссылке опасен?


Ответ: Переменная цикла остается ссылкой на последний элемент. Если не сделать `unset($item)`, следующий цикл может случайно изменить последний элемент.

Вопрос: Зачем GC, если есть reference counting?


Ответ: Reference counting не освобождает циклические ссылки. GC cycle collector находит и удаляет недостижимые циклы.

Вопрос: Почему long-running workers сложнее FPM?


Ответ: Память и state живут между jobs/requests. Утечки, static caches и request-specific данные накапливаются, поэтому нужны reset hooks, limits и observability.

### Composer

Вопрос: Разница между `composer install` и `composer update`?


Ответ: `install` ставит версии из lock-файла. `update` пересчитывает зависимости и изменяет lock-файл.

Вопрос: Нужно ли коммитить `composer.lock`?


Ответ: Для приложений да, чтобы окружения были воспроизводимыми. Для библиотек чаще нет для потребителей, но lock может быть в CI самой библиотеки.

Вопрос: Что делает `--classmap-authoritative`?


Ответ: Composer использует только classmap и не делает fallback-поиск по файловой системе. Это быстрее, но ломает динамически появляющиеся классы, если они не попали в classmap.

Вопрос: Зачем `config.platform.php`?


Ответ: Чтобы резолвить зависимости под целевую версию PHP, например production, независимо от локальной версии. Это не меняет реальный runtime.

Вопрос: В чем риск Composer scripts/plugins?


Ответ: Они выполняют код во время install/update. Нужно контролировать `allow-plugins`, не запускать недоверенный код с секретами и использовать audit/no-scripts/no-plugins при необходимости.

## Senior Checklist Перед Собеседованием

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

## Ссылки

PHP runtime и SAPI:

- [PHP Manual: Installation and Configuration](https://www.php.net/manual/en/install.php)
- [PHP Manual: Command line usage](https://www.php.net/manual/en/features.commandline.php)
- [PHP Manual: Runtime configuration](https://www.php.net/manual/en/configuration.php)
- [PHP Manual: php.ini directives](https://www.php.net/manual/en/ini.core.php)

PHP-FPM:

- [PHP Manual: FastCGI Process Manager, FPM](https://www.php.net/manual/en/install.fpm.php)
- [PHP Manual: FPM configuration](https://www.php.net/manual/en/install.fpm.configuration.php)
- [PHP Manual: FPM status page](https://www.php.net/manual/en/fpm.status.php)
- [Tideways: An introduction to PHP-FPM tuning](https://tideways.com/profiler/blog/an-introduction-to-php-fpm-tuning)

OPcache, preloading, JIT:

- [PHP Manual: OPcache](https://www.php.net/manual/en/book.opcache.php)
- [PHP Manual: OPcache configuration](https://www.php.net/manual/en/opcache.configuration.php)
- [PHP Manual: Preloading](https://www.php.net/manual/en/opcache.preloading.php)
- [PHP Manual: OPcache JIT settings](https://www.php.net/manual/en/opcache.configuration.php#ini.opcache.jit)
- [PHP.Watch: PHP 8.0 JIT](https://php.watch/versions/8.0/JIT)

Память, ссылки, GC:

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

## Короткая финальная формулировка

PHP в production чаще всего работает как пул долгоживущих FPM-процессов с изолированным request lifecycle. Производительность держится на правильном FPM sizing, контроле I/O и timeout-ов, OPcache, разумном autoload-е и воспроизводимых зависимостях. JIT и preloading полезны только после профилирования и дисциплинированного deploy. Память в PHP удобна благодаря request cleanup и copy-on-write, но references, циклы и long-running workers требуют аккуратности. Composer надо воспринимать не как простой downloader, а как dependency solver и часть supply-chain security.
