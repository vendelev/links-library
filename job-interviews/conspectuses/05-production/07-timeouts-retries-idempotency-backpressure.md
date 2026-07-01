# Timeouts, retries, idempotency, backpressure

Сквозной конспект по timeout budget, retries, idempotency, 504, long-running задачам, rate limiting и backpressure.

## Главная идея

Timeout - часть контракта между слоями. Он должен быть меньше бизнес-SLA и согласован по цепочке. Retry - distributed systems feature, а не просто `for` loop.

Senior-ответ:

> Timeout policy должна быть budget-based. Если SLA endpoint 2 секунды, нельзя дать 5 секунд на один внешний API call и еще 3 retry. Нужны общий deadline, короткие connect/read timeouts, ограниченные retries, idempotency и observability retry attempts.

## Timeout chain

Клиентские и сетевые timeout:

- DNS lookup timeout;
- TCP connect timeout;
- TLS handshake timeout;
- request write timeout;
- first byte timeout / TTFB;
- read timeout;
- idle timeout.

Nginx client-side:

- `client_header_timeout`: ожидание headers от клиента;
- `client_body_timeout`: ожидание body от клиента;
- `send_timeout`: отправка ответа клиенту;
- `keepalive_timeout`: idle keep-alive.

Nginx upstream HTTP:

- `proxy_connect_timeout`: подключение к upstream;
- `proxy_send_timeout`: отправка запроса upstream;
- `proxy_read_timeout`: ожидание ответа upstream между read operations.

Nginx FastCGI:

- `fastcgi_connect_timeout`;
- `fastcgi_send_timeout`;
- `fastcgi_read_timeout`.

PHP/app:

- `max_execution_time`;
- PHP-FPM `request_terminate_timeout`;
- DB client timeout;
- HTTP client connect/read timeout;
- queue worker timeout.

Other layers:

- CDN/WAF timeout;
- Load balancer idle/request timeout;
- service mesh timeout;
- message queue visibility timeout / ack timeout;
- object storage SDK timeout;
- browser/mobile client timeout.

## Timeout pitfalls

- Увеличение Nginx timeout не лечит медленный SQL или внешний API.
- Если app timeout больше proxy timeout, клиент получит 504, а PHP может продолжать работу.
- Если retry timeout больше общего request budget, cascade failure усиливается.
- Long-running задачи не должны жить в HTTP-запросе, их лучше отправлять в queue.
- Длинные timeouts держат соединения, воркеры, память и DB locks.
- При деградации длинные timeouts усиливают saturation.

## Budget-based design

Пример для endpoint с budget 2 секунды:

- 100 ms на routing/middleware/validation;
- 300 ms на DB;
- 700 ms на external API;
- 200 ms на Redis/cache;
- 300 ms reserve;
- 400 ms на один retry только если операция idempotent и осталось время.

Правила:

- задавать общий deadline/request budget;
- каждый downstream call получает часть budget;
- retry не должен выходить за общий deadline;
- connect timeout обычно короче read timeout;
- timeout должен быть короче пользовательского ожидания и proxy timeout;
- логировать elapsed time, retry attempt, timeout reason, dependency name.

## Retries

Retries повышают надежность при transient errors, но могут усиливать нагрузку.

Где бывают retries:

- HTTP client в приложении;
- Nginx `proxy_next_upstream`;
- Load balancer;
- Service mesh;
- Message queue consumers;
- Database clients;
- Object storage SDK;
- frontend/mobile client.

Правила:

- retry только idempotent операции или операции с idempotency key;
- использовать exponential backoff и jitter;
- ограничивать общий timeout budget;
- логировать retry attempts;
- не retry на validation/auth/business errors;
- учитывать `Retry-After` для `429`/`503`;
- не включать retries на каждом слое без общей политики.

HTTP methods:

- обычно idempotent: `GET`, `HEAD`, `PUT`, `DELETE`, `OPTIONS`;
- обычно не idempotent: `POST`, но можно сделать безопасным через idempotency key.

Pitfalls:

- retry `POST /payments` без idempotency key создает двойное списание;
- несколько уровней retries дают retry storm;
- retry на 429 без учета `Retry-After` ухудшает rate limiting;
- retry после частичного side effect может испортить данные;
- retry без trace/log поля скрывает реальную нагрузку.

## Idempotency

Idempotency означает, что повтор операции с тем же ключом не создает новый side effect.

Где нужна:

- платежи;
- создание заказов;
- webhook handling;
- queue jobs;
- external API calls;
- upload complete endpoint;
- media processing jobs;
- compensating scripts.

Практики:

- принимать `Idempotency-Key` от клиента или генерировать operation id;
- хранить key, request fingerprint, response/result, status, TTL;
- защищаться от concurrent duplicate requests;
- возвращать тот же результат для того же key;
- не переиспользовать key для другого payload;
- делать jobs идемпотентными: check current state before action.

Пример state machine:

```text
new -> processing -> succeeded
new -> processing -> failed_retryable
new -> processing -> failed_final
```

Senior-ответ:

> Для опасного `POST` я делаю idempotency на уровне бизнес-операции: key уникален для намерения пользователя, хранится вместе со статусом и результатом, а повтор возвращает тот же результат или безопасно продолжает незавершенную операцию.

## Backpressure и rate limiting

Backpressure - способ не принимать больше работы, чем система может обработать.

Инструменты:

- rate limiting;
- concurrency limits;
- queue depth limits;
- circuit breaker;
- bulkheads;
- load shedding;
- bounded retries;
- `429 Too Many Requests`;
- `503 Service Unavailable` + `Retry-After`;
- feature flag/degraded mode.

Rate limiting algorithms:

- fixed window: простой, но дает bursts на границе окна;
- sliding window: точнее, дороже;
- token bucket: разрешает burst до размера bucket;
- leaky bucket: сглаживает поток.

Где применять:

- CDN/WAF для грубой защиты;
- Nginx для IP/path-level limits;
- application для user/account/API key limits;
- queue для background workloads.

Выбор ключа:

- user ID;
- API key;
- tenant ID;
- IP как fallback;
- отдельные лимиты для login, search, export, expensive endpoints.

Pitfalls:

- лимит по IP ломает пользователей за NAT;
- без trusted proxy лимит может применяться к IP load balancer;
- без `Retry-After` клиенты не знают, когда повторить;
- rate limiting не заменяет quotas и abuse detection.

## 504 и незавершенный PHP-код

Сценарий: пользователь получил 504, а операция в базе все равно выполнилась.

Почему:

- proxy timeout истек раньше завершения PHP-кода;
- Nginx закрыл клиентское соединение;
- PHP-FPM worker продолжил выполнение;
- DB transaction или external side effect завершились позже.

Что делать:

- app-level timeout/deadline;
- cancellation, если стек это поддерживает;
- idempotency key;
- корректные транзакции;
- outbox/inbox pattern для async side effects;
- long-running operation через queue и `202 Accepted`;
- status endpoint для проверки результата.

## Long-running work

Не держать в HTTP request:

- большие exports/imports;
- media processing;
- большие uploads/downloads через PHP;
- тяжелые reports;
- массовую отправку emails;
- long DB migrations/backfills;
- внешние операции с непредсказуемой latency.

Лучший flow:

1. HTTP request валидирует и создает operation/job.
2. Возвращает `202 Accepted` и operation id.
3. Worker выполняет работу с retry/idempotency.
4. Клиент получает статус через polling/webhook/SSE.
5. Результат хранится в БД/object storage.

Для больших uploads:

- direct-to-S3 upload;
- presigned multipart;
- upload state в БД;
- checksum verification;
- background scan/processing;
- lifecycle abort incomplete multipart.

## Observability для timeout/retry политики

Логировать:

- dependency name;
- timeout type: DNS/connect/TLS/read/overall;
- duration;
- attempt number;
- max attempts;
- remaining budget;
- idempotency key или operation id;
- correlation/trace id;
- final outcome.

Метрики:

- dependency latency p50/p95/p99;
- timeout count by dependency/type;
- retry attempts;
- retry success rate;
- queue depth/lag;
- circuit breaker open count;
- 429/503 rate;
- PHP-FPM worker saturation;
- DB connection pool saturation.

Traces:

- отдельный span на external API/DB/object storage call;
- attributes: timeout, retry attempt, dependency, status;
- сохранять slow/error traces с приоритетом.

## Interview scenarios

### API иногда возвращает 504 через 60 секунд, но приложение завершает работу через 75 секунд

Разбор:

- Nginx/LB timeout меньше времени выполнения app;
- проверить `proxy_read_timeout`/`fastcgi_read_timeout`, PHP-FPM `request_terminate_timeout`, `max_execution_time`, client timeout;
- исправление не только увеличить timeout;
- вынести long-running работу в queue;
- вернуть `202 Accepted`;
- добавить idempotency и статус операции.

### Retry `POST /payments` создает двойное списание

Разбор:

- `POST` не idempotent по умолчанию;
- нужен idempotency key;
- хранить результат платежной операции;
- учитывать external provider idempotency;
- retry только при transient/network errors и в рамках budget.

### Пользователи массово получают 429

Разбор:

- проверить ключ rate limit;
- IP key может ломать NAT;
- перейти на user/API key/tenant;
- добавить `Retry-After`;
- проверить trusted proxy;
- отличать abuse от легитимного spike.

### Большие uploads приводят к 502/504 и memory pressure

Разбор:

- PHP-FPM не должен быть data plane для гигабайтных файлов;
- выдать presigned multipart upload;
- хранить upload state;
- проверять object через `HEAD`/checksum;
- scan/process в background;
- настроить lifecycle cleanup.

## Checklist

- У каждого внешнего call есть connect/read timeout.
- Есть общий request/operation deadline.
- Retry ограничен по attempts и total budget.
- Retry только для idempotent операций или с idempotency key.
- Используются exponential backoff и jitter.
- `Retry-After` учитывается для `429`/`503`.
- Long-running work вынесен из HTTP.
- Очереди имеют idempotent jobs и retry policy.
- Есть backpressure: rate/concurrency limits, circuit breaker, load shedding.
- Timeout/retry attempts видны в logs, metrics, traces.
- Proxy/app/client timeouts согласованы.
- PHP-FPM `request_terminate_timeout` согласован с Nginx и app behavior.

## Self-check

1. Чем `proxy_connect_timeout` отличается от `proxy_read_timeout`?
2. Почему 504 не обязательно отменяет PHP-код?
3. Почему нельзя просто поставить большие timeouts?
4. Когда retry безопасен?
5. Почему retry storm опасен?
6. Что такое idempotency key?
7. Как сделать `POST /payments` retry-safe?
8. Когда возвращать `202 Accepted`?
9. Чем rate limiting отличается от backpressure?
10. Почему лимит по IP плохо работает за NAT?
11. Какие метрики нужны для retry/timeout политики?
12. Почему большие uploads лучше делать direct-to-S3?

## Ссылки

- [RFC 9110: HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110)
- [MDN: HTTP response status codes](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status)
- [Nginx proxy module](https://nginx.org/en/docs/http/ngx_http_proxy_module.html)
- [Nginx FastCGI module](https://nginx.org/en/docs/http/ngx_http_fastcgi_module.html)
- [Nginx limit_req module](https://nginx.org/en/docs/http/ngx_http_limit_req_module.html)
- [AWS Builders Library: Timeouts, retries, and backoff with jitter](https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/)
- [Stripe: Idempotent requests](https://docs.stripe.com/api/idempotent_requests)
- [Google SRE: Handling Overload](https://sre.google/sre-book/handling-overload/)
- [Microsoft: Retry pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/retry)
- [Microsoft: Circuit Breaker pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/circuit-breaker)
- [Amazon S3 multipart upload](https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html)
