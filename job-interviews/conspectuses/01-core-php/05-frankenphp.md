# FrankenPHP

Цель: понимать FrankenPHP на уровне собеседования Senior/Lead PHP-разработчика: что это такое,
чем отличается от PHP-FPM, как работает worker mode, какие риски появляются в long-running PHP
и как это выводить в production.

## Документация и статьи

Официальная документация FrankenPHP:

- [FrankenPHP documentation](https://frankenphp.dev/docs/)
- [FrankenPHP GitHub](https://github.com/php/frankenphp)
- [Classic mode](https://frankenphp.dev/docs/classic/)
- [Worker mode](https://frankenphp.dev/docs/worker/)
- [Migrating from Nginx/PHP-FPM](https://frankenphp.dev/docs/migrate/)
- [Configuration](https://frankenphp.dev/docs/config/)
- [Docker images](https://frankenphp.dev/docs/docker/)
- [Deploy in production](https://frankenphp.dev/docs/production/)
- [Performance optimization](https://frankenphp.dev/docs/performance/)
- [Observability](https://frankenphp.dev/docs/observability/)
- [Logging](https://frankenphp.dev/docs/logging/)
- [Known issues](https://frankenphp.dev/docs/known-issues/)
- [Internals: architecture overview](https://frankenphp.dev/docs/internals/)
- [Static binaries](https://frankenphp.dev/docs/static/)
- [Standalone self-executable PHP apps](https://frankenphp.dev/docs/embed/)

Интеграции:

- [FrankenPHP: Symfony integration](https://frankenphp.dev/docs/symfony/)
- [FrankenPHP: Laravel integration](https://frankenphp.dev/docs/laravel/)
- [Laravel Octane: FrankenPHP](https://laravel.com/docs/octane#frankenphp)
- [Symfony Runtime component](https://symfony.com/doc/current/components/runtime.html)
- [API Platform Docker distribution](https://api-platform.com/docs/distribution/)

Смежные технологии:

- [Caddy documentation](https://caddyserver.com/docs/)
- [Caddyfile concepts](https://caddyserver.com/docs/caddyfile/concepts)
- [Caddy automatic HTTPS](https://caddyserver.com/docs/automatic-https)
- [Mercure documentation](https://mercure.rocks/docs/)
- [PHP Manual: OPcache](https://www.php.net/manual/en/book.opcache.php)
- [PHP Manual: Garbage Collection](https://www.php.net/manual/en/features.gc.php)

Статьи и доклады:

- [Kévin Dunglas: FrankenPHP, the modern PHP app server written in Go](https://dunglas.dev/2022/10/frankenphp-the-modern-php-app-server-written-in-go/)
- [Symfony blog: Introducing FrankenPHP support in Symfony Runtime](https://symfony.com/blog/introducing-frankenphp-support-in-symfony-runtime)
- [Sulu blog: Running Sulu with FrankenPHP](https://sulu.io/blog/running-sulu-with-frankenphp)
- [FrankenPHP demo and benchmarks](https://github.com/dunglas/frankenphp-demo)

## Что такое FrankenPHP

FrankenPHP - современный PHP application server, построенный поверх Caddy и написанный на Go.
Он объединяет web server, TLS, HTTP/2, HTTP/3, reverse proxy-возможности Caddy
и встроенное выполнение PHP-кода.

Ключевые возможности:

- запуск обычных PHP-приложений без Nginx + PHP-FPM;
- worker mode, где приложение загружается один раз и остается в памяти;
- автоматический HTTPS через Caddy;
- HTTP/2 и HTTP/3;
- Early Hints, real-time через Mercure, hot reload;
- Docker images и standalone/static binaries;
- интеграции с Symfony Runtime и Laravel Octane;
- метрики и структурированные логи через Caddy/FrankenPHP.

Короткий ответ для собеседования: FrankenPHP - это не фреймворк и не замена PHP-языку,
а application server/SAPI-окружение для PHP. Он может работать как более простой runtime
для обычных PHP-приложений и как high-performance long-running server через worker mode.

## Место среди PHP-FPM, RoadRunner и Swoole

PHP-FPM:

- зрелый стандарт production-деплоя;
- общается с Nginx/Apache через FastCGI;
- хорошо поддерживает shared-nothing модель;
- проще с точки зрения изоляции request state;
- требует отдельный web server и настройку FastCGI.

FrankenPHP:

- включает web server на базе Caddy;
- может заменить связку Nginx + PHP-FPM для части проектов;
- поддерживает обычный classic mode и worker mode;
- дает automatic HTTPS, HTTP/3 и Caddy middleware из коробки;
- в worker mode требует дисциплины long-running PHP.

RoadRunner:

- тоже Go-based application server;
- обычно использует отдельные PHP worker processes;
- популярен в Spiral и high-performance PHP;
- требует явной интеграции фреймворка с worker lifecycle.

Swoole/OpenSwoole:

- PHP extension с event loop, coroutines, timers, task workers;
- дает больше низкоуровневых возможностей;
- сильнее меняет модель выполнения;
- требует установки extension и аккуратного отношения к совместимости.

Senior-мысль: выбор не только про benchmark. Нужно оценивать совместимость приложения,
операционную модель, observability, поддержку командой, graceful deploy, memory leaks,
работу расширений, интеграции с фреймворком и требования платформы.

## Classic Mode

Classic mode ближе к привычной модели PHP: FrankenPHP принимает HTTP-запрос,
запускает PHP-скрипт для обработки и освобождает request-bound состояние после завершения запроса.

Пример запуска текущей директории:

```bash
frankenphp php-server
```

Пример Docker-запуска:

```bash
docker run -v .:/app/public \
    -p 80:80 -p 443:443 -p 443:443/udp \
    dunglas/frankenphp
```

Когда classic mode подходит:

- миграция с FPM без изменения mental model приложения;
- dev/staging окружения;
- приложения с legacy-кодом, где много global/static/request-specific state;
- быстрый деплой без отдельного Nginx;
- проекты, где worker mode еще не проверен под нагрузкой.

## Worker Mode

Worker mode загружает приложение один раз и держит его в памяти. На каждый HTTP-запрос вызывается handler,
а bootstrap фреймворка не повторяется полностью.

Типичный lifecycle:

1. FrankenPHP стартует worker script.
2. Composer autoload, config, DI container и framework kernel инициализируются один раз.
3. Worker ожидает запросы через `frankenphp_handle_request()`.
4. Для каждого запроса обновляются request-bound superglobals.
5. Handler обрабатывает запрос и отправляет response.
6. После response нужно выполнить terminate/reset/cleanup.
7. Worker продолжает жить или перезапускается по лимиту/ошибке/deploy.

Минимальная идея worker script:

```php
<?php

require __DIR__.'/vendor/autoload.php';

$app = new App\Kernel();
$app->boot();

$handler = static function () use ($app): void {
    try {
        echo $app->handle($_GET, $_POST, $_COOKIE, $_FILES, $_SERVER);
    } catch (Throwable $exception) {
        error_log($exception->getMessage());
    }
};

while (frankenphp_handle_request($handler)) {
    $app->terminate();
    gc_collect_cycles();
}

$app->shutdown();
```

На практике для Symfony и Laravel обычно не пишут такой script вручную. Используют Symfony Runtime
или Laravel Octane, потому что они знают, как сбрасывать framework state между запросами.

## Почему Worker Mode Быстрее

Основная экономия:

- не повторяется полный bootstrap приложения;
- DI container, routes, config и metadata уже в памяти;
- меньше операций autoload и файлового I/O;
- можно эффективнее использовать OPcache и preloaded code;
- снижается latency на CPU-bound bootstrap-heavy приложениях.

Где выигрыш заметнее:

- Laravel/Symfony монолиты с тяжелым bootstrap;
- API с большим RPS и короткой бизнес-логикой;
- приложения, где p95 latency заметно зависит от framework bootstrap;
- сервисы с хорошей stateless-дисциплиной.

Где выигрыш может быть слабым:

- запросы упираются в БД, внешние HTTP API или медленный I/O;
- приложение уже хорошо оптимизировано под FPM и OPcache;
- много тяжелой работы делается внутри каждого запроса;
- worker mode требует слишком много cleanup-кода.

## OPcache в FrankenPHP

OPcache работает в FrankenPHP, потому что FrankenPHP использует обычный PHP runtime.
Его можно настраивать через `php.ini`, Docker image или директиву `php_ini` в Caddyfile.

Пример настройки:

```caddyfile
{
    frankenphp {
        php_ini opcache.enable 1
        php_ini opcache.enable_cli 1
        php_ini opcache.memory_consumption 256
        php_ini opcache.validate_timestamps 0
    }
}
```

Зачем OPcache нужен в worker mode:

- worker всё равно стартует после deploy, reload, crash restart или достижения `max_requests`;
- при старте worker загружает Composer autoload, framework kernel, config, routes и DI container;
- OPcache снижает стоимость загрузки и компиляции PHP-кода при этих стартах;
- не весь код загружается сразу: lazy services, редкие controllers и vendor-классы могут подключаться позже;
- несколько PHP threads/workers могут переиспользовать bytecode из shared memory;
- classic mode FrankenPHP использует OPcache почти так же, как обычный PHP-FPM.

Чего OPcache не делает:

- не держит application state между запросами;
- не заменяет worker mode и не хранит готовый Laravel/Symfony kernel;
- не исправляет N+1, медленную БД, внешние API и плохие алгоритмы;
- не защищает от memory leaks и state leakage;
- не отменяет необходимость reload/restart workers после deploy.

Production-правило: для immutable deploy обычно ставят `opcache.validate_timestamps=0`
и явно перезапускают/reload-ят workers. Для dev лучше оставить timestamp validation включенной,
иначе изменения файлов могут не подхватываться без restart.

Короткий ответ для собеседования: в worker mode OPcache нужен не для каждого запроса,
а для стартов, рестартов и lazy-loaded кода. Worker mode убирает повторный bootstrap между запросами,
но OPcache снижает стоимость загрузки и компиляции PHP-кода при reload, crash recovery и `max_requests`.

## Главный Риск: State Persistence

В FPM-разработке часто неявно рассчитывают, что после запроса состояние исчезнет.
В worker mode процесс живет дальше, поэтому состояние может случайно протечь в следующий запрос.

Что может сохраняться между запросами:

- static variables внутри функций;
- class static properties;
- singleton-сервисы в DI container;
- глобальные переменные worker script;
- in-memory arrays/cache;
- открытые соединения;
- состояние сторонних библиотек;
- измененные `ini`/locale/timezone/global settings;
- часть superglobals-особенностей, например `$_ENV` не стоит использовать для request-specific данных.

Пример ошибки:

```php
final class CurrentUserContext
{
    public static ?int $userId = null;
}

CurrentUserContext::$userId = $requestUserId;
```

В обычной FPM-модели это уже плохой дизайн, но часто не приводит к межзапросной утечке.
В worker mode такой код может отдать данные одного пользователя другому запросу на том же worker thread.

Правило: все request-specific данные должны жить в request object, scoped-сервисах, explicit context,
session/token/cache/DB, но не в process-global state.

## Что Проверять Перед Worker Mode

Код приложения:

- нет request-specific данных в static/global;
- singleton-сервисы не держат `Request`, текущего пользователя, tenant, locale, correlation id;
- свои сервисы имеют reset/clear после запроса;
- временные файлы, streams и cursors закрываются;
- транзакции не остаются открытыми;
- DB/Redis/HTTP clients корректно переживают долгую жизнь процесса;
- memory usage стабилен под длительным нагрузочным тестом;
- exception handling не завершает worker неожиданно;
- shutdown/terminate hooks выполняются после response.

Фреймворк:

- для Symfony проверить Runtime FrankenPHP и resettable services;
- для Laravel использовать Octane compatibility guidance;
- не инжектить `Request` или container в долгоживущие singleton-объекты;
- проверить middleware, listeners, event subscribers и кеши;
- убедиться, что queue workers/cron/CLI не смешаны с HTTP worker state.

Production:

- настроить лимит запросов на worker/thread;
- настроить graceful restart после deploy;
- включить healthchecks;
- собрать метрики memory/RSS, restarts, crashes, queue/wait time;
- задать request timeout и upstream timeout;
- проверить rolling deploy и rollback;
- проверить поведение при fatal error и worker crash.

## Конфигурация

Пример Caddyfile для простого приложения:

```caddyfile
example.com {
    root * /var/www/app/public
    php_server
}
```

Пример worker mode:

```caddyfile
example.com {
    root * /var/www/app/public
    php_server {
        worker index.php 4
    }
}
```

Глобальные настройки FrankenPHP задаются в Caddyfile:

```caddyfile
{
    frankenphp {
        num_threads 8
        max_threads 16
        max_wait_time 30s
        max_requests 500
        php_ini memory_limit 256M
    }
}
```

Важные параметры:

- `num_threads`: стартовое количество PHP threads;
- `max_threads`: верхний лимит autoscaling threads;
- `max_wait_time`: сколько запрос может ждать свободный PHP thread;
- `max_idle_time`: когда autoscaled thread считается idle;
- `max_requests`: сколько запросов thread обработает до restart;
- `php_ini`: точечные PHP ini-настройки;
- `worker`: worker script, количество workers, env, watch paths, имя и fail policy.

## Docker и Production

Минимальный production Dockerfile из официальной идеи:

```dockerfile
FROM dunglas/frankenphp

ENV SERVER_NAME=example.com

RUN mv "$PHP_INI_DIR/php.ini-production" "$PHP_INI_DIR/php.ini"

COPY . /app
```

Минимальный compose:

```yaml
services:
  php:
    image: dunglas/frankenphp
    restart: always
    ports:
      - "80:80"
      - "443:443"
      - "443:443/udp"
    volumes:
      - caddy_data:/data
      - caddy_config:/config

volumes:
  caddy_data:
  caddy_config:
```

Production-чеклист:

- использовать `php.ini-production`;
- не монтировать исходники volume-ом в production без причины;
- хранить Caddy data/config volumes для сертификатов;
- открыть UDP `443` только если нужен HTTP/3;
- задать `SERVER_NAME` явно;
- настроить logs в stdout/stderr для контейнерной платформы;
- pin-ить версию image, если важна воспроизводимость;
- отдельно продумать build PHP extensions;
- проверять compatibility extension-ов с ZTS/threaded окружением;
- использовать process supervisor/orchestrator restart policy.

## Laravel

Laravel обычно запускают через Octane с сервером FrankenPHP.

Установка:

```bash
composer require laravel/octane
php artisan octane:install --server=frankenphp
```

Запуск:

```bash
php artisan octane:start --server=frankenphp
```

FrankenPHP-specific команда из документации:

```bash
frankenphp php-cli artisan octane:frankenphp
```

Главные риски Laravel Octane:

- singleton держит старый `Request`;
- singleton держит старый container/config/request-specific dependency;
- static cache растет между запросами;
- service provider `register`/`boot` выполняется один раз на worker;
- изменение `.env`, routes, config и code требует reload;
- package не готов к long-running lifecycle.

Senior-answer: Octane ускоряет Laravel за счет boot once/run many, но платой становится необходимость
писать Octane-friendly код. Нельзя blindly включить Octane/FrankenPHP на legacy-монолите
без нагрузочного теста и аудита stateful-сервисов.

## Symfony

Symfony использует Runtime component и FrankenPHP Runtime.

Пример Caddyfile:

```caddyfile
localhost

root public/
php_server {
    worker ./public/index.php
}
```

Пример Docker-запуска worker mode:

```bash
docker run \
    -e FRANKENPHP_CONFIG="worker ./public/index.php" \
    -e APP_RUNTIME=Runtime\FrankenPhpSymfony\Runtime \
    -v $PWD:/app \
    -p 80:80 -p 443:443 -p 443:443/udp \
    dunglas/frankenphp
```

На что смотреть в Symfony:

- services, реализующие `Symfony\Contracts\Service\ResetInterface`;
- request stack и request-scoped данные;
- event listeners/subscribers с mutable state;
- Doctrine EntityManager lifecycle;
- Messenger workers отдельно от HTTP workers;
- profiler/debug tooling в dev и production.

## Observability

Что важно видеть:

- request count, latency, p95/p99;
- active/idle PHP threads;
- queue/wait time до свободного thread;
- worker crashes и restarts;
- memory/RSS по процессу;
- `max_requests` restarts;
- HTTP status distribution;
- upstream DB/Redis/HTTP latency;
- GC pauses и рост heap;
- deploy/reload events.

FrankenPHP предоставляет Prometheus-метрики для threads, workers, queues, crashes и restarts.
Логи идут через Caddy; можно использовать `error_log()` и специальные logging-возможности FrankenPHP.

Практический подход:

- сравнить baseline FPM и FrankenPHP на одинаковом workload;
- гонять soak test, а не только короткий benchmark;
- проверять рост памяти на тысячах/миллионах запросов;
- смотреть не только RPS, но и tail latency;
- отдельно тестировать deploy/reload/failure сценарии.

## Performance Trade-Offs

Что может ускориться:

- bootstrap-heavy HTTP requests;
- route/config/container initialization;
- короткие API-запросы;
- TLS/HTTP stack за счет Caddy;
- static files и modern HTTP features.

Что может стать хуже:

- memory footprint, если держать много workers с тяжелым container;
- утечки памяти становятся накопительными;
- сложнее локализовать state leakage bugs;
- часть extensions/packages может плохо вести себя в threaded/long-running режиме;
- deploy требует reload workers, иначе старый код остается в памяти.

Важная формулировка: worker mode улучшает latency не магически,
а потому что убирает повторяющуюся инициализацию. Если bottleneck в БД, N+1 запросах,
блокирующих API или плохих индексах, FrankenPHP это не исправит.

## Частые Ошибки

- Включить worker mode без аудита static/global state.
- Считать FrankenPHP полной заменой архитектурной оптимизации.
- Оценивать только RPS без p95/p99 и memory growth.
- Не настроить worker restart/max requests.
- Не перезапускать workers после deploy.
- Хранить request/user/tenant в singleton service.
- Игнорировать совместимость PHP extensions.
- Оставить dev hot reload/watch в production.
- Не настроить timeouts и healthchecks.
- Сравнивать FrankenPHP и FPM при разных `php.ini`, OPcache и hardware settings.

## Как Отвечать На Собеседовании

Вопрос: что такое FrankenPHP?

Ответ: это PHP application server на базе Caddy, который может обслуживать PHP-приложения
без отдельного Nginx + PHP-FPM и поддерживает worker mode. В worker mode приложение boot-ится один раз
и обрабатывает много запросов, поэтому снижается overhead bootstrap,
но появляется риск persistence state между запросами.

Вопрос: чем worker mode отличается от FPM?

Ответ: в FPM worker process тоже живет дольше одного запроса, но прикладной request state обычно
пересоздается и очищается моделью выполнения. В FrankenPHP worker mode сам framework/application kernel
остается в памяти, поэтому static properties, singletons, in-memory cache и mutable services
могут переживать запросы.

Вопрос: как безопасно мигрировать?

Ответ: начать с classic mode или staging worker mode, провести аудит request-specific state,
включить max requests, metrics, healthchecks, проверить framework integration,
сделать нагрузочный и soak test, сравнить p95/p99 и memory growth с FPM, затем выкатывать постепенно.

Вопрос: когда не стоит использовать worker mode?

Ответ: когда приложение legacy-heavy, много static/global state, нет тестов,
неизвестна совместимость packages/extensions, bottleneck в БД/I/O,
команда не готова сопровождать long-running PHP или нет observability для memory leaks и worker restarts.

Вопрос: почему нужен `max_requests`?

Ответ: PHP и многие библиотеки исторически проектировались под request lifecycle.
В long-running worker даже небольшая утечка или накопление state со временем растет.
`max_requests` периодически перезапускает worker/thread и ограничивает ущерб.

## Мини-Практика

Разбор кода:

```php
final class TenantContext
{
    private static ?string $tenantId = null;

    public static function set(string $tenantId): void
    {
        self::$tenantId = $tenantId;
    }

    public static function get(): ?string
    {
        return self::$tenantId;
    }
}
```

Что сказать на собеседовании:

- В worker mode `TenantContext::$tenantId` может пережить запрос.
- Следующий запрос на том же worker может увидеть чужой tenant.
- Даже если в конце middleware есть cleanup, исключение или early return может его пропустить.
- Лучше передавать tenant через request-scoped context или сервис, который framework гарантированно reset-ит.
- Нужно добавить тест/soak scenario, который делает два запроса с разными tenants на один worker.

## Финальный Чеклист

- Могу объяснить, что FrankenPHP построен поверх Caddy.
- Могу отличить classic mode от worker mode.
- Понимаю, почему worker mode быстрее на bootstrap-heavy приложениях.
- Понимаю, какие состояния сохраняются между запросами.
- Могу назвать риски static/global/singleton/request injection.
- Могу объяснить `max_requests`, graceful reload и healthchecks.
- Могу описать production Docker/Caddyfile базу.
- Знаю, что Laravel идет через Octane, Symfony через Runtime.
- Не обещаю performance win без benchmark и soak test.
- Умею сравнить FrankenPHP с PHP-FPM, RoadRunner и Swoole.
