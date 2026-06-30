# HTTP, DNS, TLS, Nginx, reverse proxy, timeouts

Гайд для подготовки к интервью Lead/Senior PHP backend developer. Фокус: как запрос проходит от браузера до PHP-кода, где возникают задержки и ошибки, как объяснять решения на уровне production-систем.

## Карта запроса

Типовой путь запроса:

1. Клиент получает IP через DNS.
2. Клиент открывает TCP-соединение или использует существующее keep-alive-соединение.
3. Для HTTPS выполняется TLS-handshake.
4. Клиент отправляет HTTP-запрос.
5. Reverse proxy, например Nginx, принимает запрос.
6. Nginx выбирает upstream: приложение, балансировщик, PHP-FPM через FastCGI, CDN или другой сервис.
7. PHP-FPM исполняет PHP-код, обращается к БД, кешу, очередям, внешним API.
8. Ответ проходит обратно через Nginx и сеть к клиенту.

Senior-ответ: важно уметь разложить latency по этапам: DNS, connect, TLS, TTFB, upstream processing, download. Без этого диагностика 502/504/499 превращается в угадывание.

## DNS basics

DNS преобразует имя домена в IP-адрес и другие записи.

Основные записи:

- `A`: IPv4-адрес.
- `AAAA`: IPv6-адрес.
- `CNAME`: алиас на другое DNS-имя.
- `MX`: почтовые серверы.
- `TXT`: произвольные текстовые записи, часто SPF/DKIM/DMARC, верификации.
- `NS`: authoritative name servers зоны.
- `CAA`: какие CA могут выпускать TLS-сертификаты для домена.

Ключевые понятия:

- Recursive resolver: резолвер провайдера, Google DNS, Cloudflare DNS, корпоративный DNS.
- Authoritative DNS: источник истины для зоны.
- TTL: время кеширования записи.
- Negative caching: кеширование отсутствия записи, например `NXDOMAIN`.
- Split-horizon DNS: разные ответы для внутренних и внешних клиентов.

Pitfalls:

- Слишком высокий TTL замедляет миграции и аварийное переключение.
- Слишком низкий TTL не гарантирует мгновенный failover: клиенты и резолверы могут кешировать дольше.
- `CNAME` на apex-домене обычно нельзя использовать в классическом DNS, нужны `ALIAS`/`ANAME` у провайдера.
- IPv6 может ломать доступ, если `AAAA` указывает на неработающий endpoint.
- DNS round-robin не является полноценным load balancing: нет health checks на стороне DNS-клиента.

Senior-ответ: DNS не знает о состоянии приложения, а TTL не является SLA на переключение. Для надежного failover нужны health checks, балансировщики, anycast/CDN или управляемый DNS с проверками.

## TCP и TLS

TCP дает надежную доставку байтов поверх IP.

Важные свойства TCP:

- 3-way handshake: `SYN`, `SYN-ACK`, `ACK`.
- Порядок доставки и retransmission.
- Flow control и congestion control.
- Соединение имеет состояние на клиенте, сервере и промежуточных устройствах.
- Закрытие через `FIN` или аварийный `RST`.

TLS обеспечивает шифрование, аутентификацию сервера и целостность данных.

TLS handshake включает:

- согласование версии TLS и cipher suite;
- проверку сертификата и цепочки доверия;
- обмен ключами;
- после handshake HTTP идет внутри зашифрованного канала.

TLS 1.3 обычно быстрее TLS 1.2: меньше round trips, лучше современные cipher suites, есть session resumption.

Pitfalls:

- Просроченный сертификат или неполная цепочка ломают клиентов.
- Неправильный SNI приводит к выдаче не того сертификата.
- Старые клиенты могут не поддерживать современные настройки TLS.
- OCSP/CRL и проверка сертификатов могут добавлять latency.
- Termination TLS на reverse proxy означает, что дальше до upstream может идти HTTP или отдельный TLS.

Senior-ответ: HTTPS-ошибка не всегда про приложение. Нужно проверять DNS, SNI, цепочку сертификата, ALPN, TLS-версию, cipher suite, termination point и настройки proxy-to-upstream.

## HTTP/1.1 vs HTTP/2 vs HTTP/3

HTTP/1.1:

- текстовый протокол;
- один запрос активно обрабатывается на соединении в конкретный момент, pipelining почти не используется;
- keep-alive позволяет переиспользовать TCP/TLS-соединение;
- head-of-line blocking на уровне соединения.

HTTP/2:

- бинарный протокол;
- мультиплексирование нескольких stream поверх одного TCP-соединения;
- HPACK-сжатие заголовков;
- server push устарел и практически не рекомендуется;
- head-of-line blocking TCP все еще возможен при потере пакетов.

HTTP/3:

- работает поверх QUIC/UDP;
- TLS 1.3 встроен в QUIC;
- меньше проблем с TCP head-of-line blocking между stream;
- лучше переносит смену сети, например Wi-Fi на LTE;
- требует поддержки на клиенте, edge/proxy и сетевой инфраструктуре.

Pitfalls:

- HTTP/2 не ускоряет медленный backend сам по себе.
- Один HTTP/2 connection может концентрировать много запросов, поэтому настройки limits важны.
- Между клиентом и Nginx может быть HTTP/2, а между Nginx и PHP-FPM не HTTP, а FastCGI.
- HTTP/3 может блокироваться сетями, где UDP ограничен.

Senior-ответ: версия HTTP влияет на сетевой overhead и конкуренцию запросов, но не заменяет оптимизацию backend latency, кеширования и лимитов upstream.

## Keep-Alive

Keep-alive переиспользует соединение для нескольких запросов.

Плюсы:

- меньше TCP/TLS handshakes;
- ниже latency;
- меньше CPU на TLS;
- лучше throughput.

Минусы и риски:

- idle-соединения занимают память и file descriptors;
- слишком длинный keep-alive может держать ресурсы без пользы;
- несовпадение timeout между proxy, load balancer и app вызывает обрывы;
- connection reuse может проявлять проблемы с upstream, если stale connections не обрабатываются.

Nginx-настройки, которые часто обсуждают:

- `keepalive_timeout`: сколько держать idle client connection.
- `keepalive_requests`: сколько запросов разрешить на одном соединении.
- `upstream keepalive`: пул keep-alive соединений к HTTP upstream.

Senior-ответ: keep-alive полезен, но timeout-цепочка должна быть согласована. Обычно внешний load balancer idle timeout должен быть больше или согласован с Nginx, а upstream timeouts должны соответствовать реальному SLA backend.

## HTTP headers

Заголовки передают метаданные запроса и ответа.

Частые request headers:

- `Host`: виртуальный хост.
- `User-Agent`: клиент.
- `Accept`, `Accept-Encoding`, `Accept-Language`: согласование формата, сжатия, языка.
- `Authorization`: credentials.
- `Cookie`: cookies клиента.
- `Content-Type`: тип тела запроса.
- `Content-Length`: размер тела.
- `X-Forwarded-For`: исходный IP через proxy.
- `X-Forwarded-Proto`: исходная схема `http`/`https`.
- `Forwarded`: стандартизированная альтернатива `X-Forwarded-*`.

Частые response headers:

- `Content-Type`: тип ответа.
- `Content-Length` или chunked transfer.
- `Cache-Control`, `ETag`, `Last-Modified`.
- `Set-Cookie`.
- `Location` для redirects.
- `Vary`: какие request headers влияют на вариант ответа.
- `Strict-Transport-Security`.
- `Content-Security-Policy`.

Pitfalls:

- Нельзя доверять `X-Forwarded-For` от клиента без настройки trusted proxies.
- Отсутствие `Vary: Origin` при CORS и кешировании может отдавать неверные ответы разным origin.
- `Content-Length` должен соответствовать body, иначе возможны hanging requests или protocol errors.
- Заголовки могут быть ограничены по размеру в Nginx, CDN, LB и PHP.

Senior-ответ: headers являются частью security boundary. Особенно важно корректно обрабатывать forwarded headers, cookies, auth headers и cache headers.

## Caching headers

Основные заголовки кеширования:

- `Cache-Control`: главная политика кеширования.
- `Expires`: устаревший, но все еще поддерживаемый absolute expiry.
- `ETag`: validator версии ресурса.
- `Last-Modified`: validator по времени изменения.
- `Vary`: ключ варианта кеша.
- `Age`: сколько ответ уже провел в shared cache.

Полезные директивы `Cache-Control`:

- `no-store`: не сохранять нигде.
- `no-cache`: можно сохранить, но перед использованием нужно revalidate.
- `private`: только private cache, например браузер.
- `public`: можно shared cache.
- `max-age=SECONDS`: свежесть для клиента.
- `s-maxage=SECONDS`: свежесть для shared cache.
- `must-revalidate`: нельзя использовать stale без проверки.
- `immutable`: ресурс не изменится во время freshness lifetime.

Стратегии:

- Static assets: `Cache-Control: public, max-age=31536000, immutable` с fingerprint в имени файла.
- HTML/API user-specific: часто `private, no-store` или короткий `max-age`.
- Public API lists: `ETag`/`Last-Modified` плюс аккуратный `Vary`.

Pitfalls:

- Кеширование персональных данных через shared cache.
- Отсутствие `Vary: Authorization` или запрет кеша для auth-запросов.
- Использование `no-cache` как запрета сохранения. Для полного запрета нужен `no-store`.
- Кеш без invalidation-стратегии становится источником старых данных.

Senior-ответ: хороший cache design начинается с классификации данных: public/private, mutable/immutable, user-specific/shared, допустимая stale window.

## Cookies

Cookie хранит небольшие данные на клиенте и отправляется на подходящие запросы.

Важные атрибуты:

- `HttpOnly`: недоступна из JavaScript.
- `Secure`: отправляется только по HTTPS.
- `SameSite=Lax|Strict|None`: контроль cross-site отправки.
- `Domain`: область доменов.
- `Path`: область путей.
- `Expires`/`Max-Age`: срок жизни.

Security practices:

- Session cookie: `HttpOnly; Secure; SameSite=Lax` как базовая настройка.
- Для third-party/cross-site cookie нужен `SameSite=None; Secure`.
- Не хранить секреты в cookie без подписи/шифрования.
- Минимизировать размер cookie: она отправляется на каждый подходящий запрос.

Pitfalls:

- Большие cookies увеличивают latency и могут превысить лимиты headers.
- Неверный `Domain` может открыть cookie поддоменам.
- `SameSite=None` без `Secure` отклоняется современными браузерами.
- Cookie-based auth требует защиты от CSRF.

Senior-ответ: cookie - это транспорт state между браузером и сервером, а не безопасное хранилище само по себе. Без `HttpOnly`, `Secure`, `SameSite` и CSRF-модели легко получить уязвимость.

## CORS

CORS ограничивает, какие browser origins могут читать ответы cross-origin запросов.

Основные заголовки:

- `Access-Control-Allow-Origin`.
- `Access-Control-Allow-Methods`.
- `Access-Control-Allow-Headers`.
- `Access-Control-Allow-Credentials`.
- `Access-Control-Max-Age`.

Preflight:

- Браузер отправляет `OPTIONS` перед небезопасным cross-origin запросом.
- Сервер должен ответить разрешенными методами и заголовками.
- Preflight можно кешировать через `Access-Control-Max-Age`.

Pitfalls:

- `Access-Control-Allow-Origin: *` нельзя использовать вместе с credentials.
- CORS не защищает backend от server-to-server запросов.
- CORS - browser security mechanism, не authentication и не authorization.
- При dynamic origin нужен `Vary: Origin`.
- Redirect во время preflight часто ломает запрос.

Senior-ответ: CORS решает только вопрос, разрешено ли браузеру читать ответ. Backend все равно обязан проверять auth, permissions, CSRF и input validation.

## Reverse proxy

Reverse proxy принимает запросы от клиентов и проксирует их к backend.

Задачи reverse proxy:

- TLS termination.
- Routing по host/path/header.
- Load balancing.
- Static files.
- Compression.
- Request/response buffering.
- Rate limiting.
- Access logs.
- Header normalization.
- Security headers.

Nginx как reverse proxy часто стоит перед:

- PHP-FPM через FastCGI;
- HTTP backend, например Symfony/Laravel Octane, Swoole, RoadRunner, Node.js;
- API gateway;
- internal services.

Pitfalls:

- Потеря исходного IP без `X-Forwarded-For` и trusted proxy config.
- Неверный `X-Forwarded-Proto` ломает redirects, secure cookies и генерацию URL.
- Дублирование или конфликт timeout между CDN, LB, Nginx и app.
- Proxy buffering может скрывать streaming и SSE.

Senior-ответ: reverse proxy - это не просто pipe. Он меняет поведение приложения через headers, buffering, timeouts, body limits и retry policy.

## Load balancing

Алгоритмы балансировки:

- Round-robin: равномерное распределение по списку upstream.
- Least connections: меньше активных соединений.
- IP hash: sticky routing по IP.
- Hash by key: sticky routing по cookie/header/URI.
- Weighted: разные веса для upstream.

Health checks:

- Passive: upstream помечается failed после ошибок.
- Active: балансировщик сам проверяет endpoint.

Риски sticky sessions:

- сложнее масштабирование;
- деградация при падении узла;
- state лучше хранить во внешнем хранилище: Redis, DB, shared session storage.

Nginx open source имеет passive health checks. Active health checks доступны в Nginx Plus или реализуются внешними средствами.

Pitfalls:

- Retry небезопасных методов может создать дублирующие операции.
- Балансировка по IP плохо работает за NAT и мобильными сетями.
- Нет смысла балансировать на мертвые upstream без health checks.
- Разные версии приложения за балансировщиком требуют совместимости схемы данных и API.

Senior-ответ: load balancing должен учитывать idempotency, health checks, graceful deploys, connection draining и observability по каждому upstream.

## Nginx + PHP-FPM/FastCGI

PHP-FPM - process manager для PHP. Nginx передает PHP-запросы в FPM через FastCGI.

Типовая схема:

```nginx
location ~ \.php$ {
    include fastcgi_params;
    fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    fastcgi_pass unix:/run/php/php-fpm.sock;
}
```

Для фронт-контроллера чаще используют:

```nginx
location / {
    try_files $uri /index.php$is_args$args;
}

location ~ ^/index\.php(/|$) {
    include fastcgi_params;
    fastcgi_param SCRIPT_FILENAME $realpath_root/index.php;
    fastcgi_param DOCUMENT_ROOT $realpath_root;
    fastcgi_pass unix:/run/php/php-fpm.sock;
    internal;
}
```

Важные параметры:

- `fastcgi_pass`: socket или TCP endpoint FPM.
- `fastcgi_param SCRIPT_FILENAME`: путь до исполняемого PHP-файла.
- `fastcgi_read_timeout`: ожидание ответа от FPM.
- `fastcgi_connect_timeout`: подключение к FPM.
- `fastcgi_send_timeout`: отправка запроса в FPM.
- `fastcgi_buffers`, `fastcgi_buffer_size`: буферизация ответа.

PHP-FPM параметры:

- `pm = dynamic|static|ondemand`.
- `pm.max_children`: максимум воркеров.
- `pm.max_requests`: перезапуск воркера после N запросов.
- `request_terminate_timeout`: жесткое ограничение времени PHP-запроса.
- `slowlog`: лог медленных запросов.
- `listen.backlog`: очередь подключений к FPM.

Pitfalls:

- Неверный `SCRIPT_FILENAME` может привести к 404, 502 или security issue.
- `pm.max_children` слишком мал: очередь, 502/504, рост latency.
- `pm.max_children` слишком велик: memory exhaustion и swap.
- `request_terminate_timeout` должен быть согласован с Nginx timeout.
- `clear_env` влияет на доступность env-переменных.

Senior-ответ: для PHP-FPM capacity planning считают память одного PHP worker под реальной нагрузкой, затем выбирают `pm.max_children`, чтобы не уходить в swap. CPU, DB pool и external calls тоже ограничивают throughput.

## Buffering

Nginx может буферизовать request body и response body.

Request buffering:

- Nginx читает тело запроса от клиента до передачи upstream.
- Защищает backend от slow clients.
- Может писать большие тела во временные файлы.

Response buffering:

- Nginx читает ответ upstream в буферы и отдает клиенту своим темпом.
- Освобождает upstream быстрее при медленном клиенте.
- Может мешать streaming, SSE и long polling.

Важные директивы:

- `proxy_request_buffering` / `fastcgi_request_buffering`.
- `proxy_buffering` / `fastcgi_buffering`.
- `client_body_buffer_size`.
- `proxy_buffers`, `fastcgi_buffers`.
- `X-Accel-Buffering: no` для отключения buffering в отдельных ответах.

Pitfalls:

- Streaming не работает, потому что proxy buffering включен.
- Большие upload/download создают временные файлы и I/O pressure.
- Отключение request buffering передает slow client проблему backend.

Senior-ответ: buffering - это tradeoff между защитой upstream, memory/disk usage и возможностью streaming. Его нельзя включать или выключать глобально без понимания endpoint semantics.

## Gzip и Brotli

Сжатие уменьшает размер ответа, но тратит CPU.

Gzip:

- широко поддерживается;
- встроен в Nginx;
- хороший baseline.

Brotli:

- часто дает лучшее сжатие для text assets;
- требует модуля Nginx или поддержки на edge/CDN;
- особенно полезен для precompressed static assets.

Что сжимать:

- HTML, CSS, JS, JSON, SVG, XML.

Что обычно не сжимать:

- JPEG, PNG, WebP, AVIF, MP4, ZIP, PDF с внутренним сжатием.

Pitfalls:

- Сжатие маленьких ответов может быть дороже пользы.
- Сжатие уже сжатых файлов бесполезно.
- Нужно учитывать `Accept-Encoding` и `Vary: Accept-Encoding`.
- Dynamic compression под высокой нагрузкой может съедать CPU.

Senior-ответ: для static assets лучше precompressed `.br`/`.gz` плюс fingerprint caching. Для dynamic API нужно измерять CPU cost и latency.

## Upload limits

Ограничения загрузки могут быть на нескольких уровнях:

- CDN/WAF.
- Load balancer.
- Nginx `client_max_body_size`.
- PHP `post_max_size`.
- PHP `upload_max_filesize`.
- Application validation.
- Disk/temp storage.

Типовые симптомы:

- `413 Payload Too Large` от Nginx или CDN.
- Пустой `$_POST`/`$_FILES`, если `post_max_size` превышен.
- 502/504 при слишком долгой обработке upload.
- Disk full из-за временных файлов.

Pitfalls:

- Увеличили `upload_max_filesize`, но забыли `post_max_size`.
- Увеличили PHP limits, но Nginx `client_max_body_size` остался меньше.
- Upload идет через PHP, хотя лучше прямой upload в object storage по signed URL.
- Нет лимита на количество файлов и content-type validation.

Senior-ответ: большие файлы лучше выгружать напрямую в object storage, а backend должен выдавать signed URL и обрабатывать metadata/asynchronous scanning.

## Timeouts

Timeout - это часть контракта между слоями. Он должен быть меньше бизнес-SLA и согласован по цепочке.

Клиентские и сетевые timeout:

- DNS lookup timeout.
- TCP connect timeout.
- TLS handshake timeout.
- Request write timeout.
- First byte timeout / TTFB.
- Read timeout.
- Idle timeout.

Nginx client-side:

- `client_header_timeout`: ожидание headers от клиента.
- `client_body_timeout`: ожидание body от клиента.
- `send_timeout`: отправка ответа клиенту.
- `keepalive_timeout`: idle keep-alive.

Nginx upstream HTTP:

- `proxy_connect_timeout`: подключение к upstream.
- `proxy_send_timeout`: отправка запроса upstream.
- `proxy_read_timeout`: ожидание ответа upstream между read operations.

Nginx FastCGI:

- `fastcgi_connect_timeout`.
- `fastcgi_send_timeout`.
- `fastcgi_read_timeout`.

PHP:

- `max_execution_time`.
- PHP-FPM `request_terminate_timeout`.
- DB client timeout.
- HTTP client connect/read timeout.
- Queue worker timeout.

Pitfalls:

- Увеличение Nginx timeout не лечит медленный SQL или внешний API.
- Если app timeout больше proxy timeout, клиент получит 504, а PHP может продолжать работу.
- Если retry timeout больше общего request budget, cascade failure усиливается.
- Long-running задачи не должны жить в HTTP-запросе, их лучше отправлять в queue.

Senior-ответ: timeout policy должна быть budget-based. Например, если SLA endpoint 2 секунды, нельзя дать 5 секунд на один внешний API call и еще 3 retry.

## Retries

Retries повышают надежность при transient errors, но могут усиливать нагрузку.

Где бывают retries:

- HTTP client в приложении.
- Nginx `proxy_next_upstream`.
- Load balancer.
- Service mesh.
- Message queue consumers.
- Database clients.

Правила:

- Retry только idempotent операции или операции с idempotency key.
- Использовать exponential backoff и jitter.
- Ограничивать общий timeout budget.
- Логировать retry attempts.
- Не retry на явные validation/auth errors.

HTTP методы:

- Обычно idempotent: `GET`, `HEAD`, `PUT`, `DELETE`, `OPTIONS`.
- Обычно не idempotent: `POST`, но можно сделать безопасным через idempotency key.

Pitfalls:

- Retry `POST /payments` без idempotency key создает двойное списание.
- Несколько уровней retries дают retry storm.
- Retry на 429 без учета `Retry-After` ухудшает rate limiting.

Senior-ответ: retry - это distributed systems feature, а не просто `for` loop. Нужны idempotency, backoff, jitter, observability и общий timeout budget.

## Status codes

Группы:

- `1xx`: informational.
- `2xx`: success.
- `3xx`: redirection.
- `4xx`: client error.
- `5xx`: server error.

Частые коды:

- `200 OK`: успешный ответ.
- `201 Created`: ресурс создан.
- `202 Accepted`: принято в асинхронную обработку.
- `204 No Content`: успешно, тела нет.
- `301 Moved Permanently`: постоянный redirect.
- `302 Found`: временный redirect, исторически меняет метод в браузерах.
- `303 See Other`: redirect на GET после POST.
- `307 Temporary Redirect`: временный redirect с сохранением метода.
- `308 Permanent Redirect`: постоянный redirect с сохранением метода.
- `400 Bad Request`: некорректный запрос.
- `401 Unauthorized`: нужна authentication.
- `403 Forbidden`: authentication есть или не нужна, но доступ запрещен.
- `404 Not Found`: не найдено.
- `409 Conflict`: конфликт состояния.
- `412 Precondition Failed`: не прошли preconditions, например `If-Match`.
- `413 Payload Too Large`: тело слишком большое.
- `415 Unsupported Media Type`: неверный `Content-Type`.
- `422 Unprocessable Content`: семантическая ошибка валидации.
- `429 Too Many Requests`: rate limit.
- `500 Internal Server Error`: ошибка сервера.
- `502 Bad Gateway`: proxy получил плохой ответ от upstream.
- `503 Service Unavailable`: сервис недоступен или перегружен.
- `504 Gateway Timeout`: proxy не дождался upstream.

Nginx-specific в логах:

- `499`: клиент закрыл соединение до ответа. Это не стандартный HTTP-код.

Pitfalls:

- Путать `401` и `403`.
- Возвращать `500` на validation errors.
- Использовать `200` с error body для ошибок API.
- Не отдавать `Retry-After` при `429`/`503`, когда клиенту полезно знать паузу.

Senior-ответ: status code - часть API-контракта. Он влияет на кеши, retries, clients, observability и incident response.

## Rate limiting

Rate limiting защищает от abuse, spikes и случайных перегрузок.

Алгоритмы:

- Fixed window: простой, но дает bursts на границе окна.
- Sliding window: точнее, дороже.
- Token bucket: разрешает burst до размера bucket.
- Leaky bucket: сглаживает поток.

Где применять:

- CDN/WAF для грубой защиты.
- Nginx для IP/path-level limits.
- Application для user/account/API key limits.
- Queue для background workloads.

Nginx пример:

```nginx
limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;

server {
    location /api/ {
        limit_req zone=api burst=20 nodelay;
        proxy_pass http://backend;
    }
}
```

Pitfalls:

- Лимит по IP ломает пользователей за NAT.
- Нужно учитывать trusted proxy, иначе лимит применяется к IP load balancer.
- Без `Retry-After` клиенты не знают, когда повторить запрос.
- Rate limiting не заменяет quotas и abuse detection.

Senior-ответ: хороший rate limit имеет правильный ключ: user ID, API key, tenant ID, IP как fallback. Нужны разные лимиты для login, search, export, expensive endpoints.

## Troubleshooting 499, 502, 504

### 499 Client Closed Request

Что значит:

- Клиент закрыл соединение до того, как Nginx отдал ответ.
- Это может быть браузер, мобильное приложение, CDN, load balancer или timeout клиента.

Частые причины:

- Backend отвечает дольше client timeout.
- Пользователь отменил запрос или ушел со страницы.
- CDN/LB оборвал соединение раньше Nginx.
- Мобильная сеть нестабильна.
- Большой response download, клиент не дождался.

Что проверять:

- `$request_time` в Nginx access log.
- `$upstream_response_time`.
- user agent, endpoint, размер ответа.
- timeouts на CDN/LB/client.
- backend latency и slow logs.

Senior-ответ: 499 часто симптом медленного backend или короткого client timeout, но не всегда ошибка сервера. Сначала надо коррелировать request time, upstream time и клиентский контекст.

### 502 Bad Gateway

Что значит:

- Nginx не смог получить корректный ответ от upstream.

Частые причины PHP-FPM:

- PHP-FPM не запущен.
- Неверный socket/path/permissions.
- Все FPM workers заняты, backlog переполнен.
- PHP process crashed или был убит OOM killer.
- Неверный FastCGI response.
- Слишком большой header от upstream.

Что проверять:

- Nginx error log: `connect() failed`, `upstream prematurely closed connection`, `recv() failed`, `upstream sent too big header`.
- Статус PHP-FPM и pool logs.
- `pm.max_children reached`.
- OOM killer logs.
- socket permissions.
- deploy/restart события.

Senior-ответ: 502 - это проблема связи proxy-upstream или некорректного upstream response. Для PHP первым делом смотрю Nginx error log и PHP-FPM pool log, потом capacity FPM и memory.

### 504 Gateway Timeout

Что значит:

- Nginx подключился к upstream, но не дождался ответа в пределах read timeout.

Частые причины:

- Медленный SQL.
- Lock contention.
- Внешний API завис.
- Очередь в PHP-FPM.
- Deadlock или бесконечная операция.
- Неправильно выбран timeout для long-running endpoint.

Что проверять:

- `$upstream_response_time` близок к `fastcgi_read_timeout`/`proxy_read_timeout`.
- PHP slowlog.
- DB slow query log.
- traces/APM.
- saturation PHP-FPM workers.
- external dependency latency.

Senior-ответ: 504 не лечится механическим увеличением timeout. Сначала нужно понять, где потрачено время: очередь FPM, PHP execution, SQL, network call или lock.

## Практический алгоритм диагностики

1. Проверить, воспроизводится ли проблема и на каком endpoint.
2. Посмотреть access log: status, `$request_time`, `$upstream_response_time`, bytes, user agent.
3. Посмотреть Nginx error log по request id или времени.
4. Проверить upstream health: PHP-FPM status, workers, backlog, CPU, memory, OOM.
5. Проверить application logs и slow logs.
6. Проверить DB/cache/external dependencies.
7. Сравнить timeouts по цепочке: client, CDN, LB, Nginx, PHP-FPM, app clients.
8. Сформулировать fix: capacity, query optimization, timeout budget, queue, caching, retry/idempotency, rate limit.

Полезные log variables Nginx:

- `$request_time`: полное время обработки запроса в Nginx.
- `$upstream_response_time`: время ответа upstream.
- `$upstream_connect_time`: время подключения к upstream.
- `$upstream_header_time`: время до headers от upstream.
- `$status`: финальный статус.
- `$upstream_status`: статус от upstream.
- `$request_length`: размер запроса.
- `$bytes_sent`: отправлено клиенту.
- `$http_x_forwarded_for`: forwarded client IP.

## Типовые senior-вопросы и ответы

Вопрос: почему пользователь получил 504, а операция в базе все равно выполнилась?

Ответ: proxy timeout истек раньше завершения PHP-кода или DB operation. Nginx закрыл клиентский запрос, но PHP/FPM мог продолжить выполнение. Нужны app-level timeout/cancellation, idempotency key, транзакции и корректный timeout budget.

Вопрос: почему после включения HTTP/2 backend не стал быстрее?

Ответ: HTTP/2 уменьшает overhead соединений и помогает multiplexing на участке client-proxy, но если latency в PHP, SQL или внешнем API, bottleneck не изменится.

Вопрос: почему `X-Forwarded-For` опасен?

Ответ: клиент может прислать этот header сам. Доверять ему можно только от trusted proxies, иначе можно обойти IP-based ACL, rate limit или audit.

Вопрос: когда отключать proxy buffering?

Ответ: для streaming/SSE/long polling, где клиент должен получать chunks сразу. Для обычных API buffering полезен, потому что защищает upstream от slow clients.

Вопрос: чем отличается 502 от 504?

Ответ: 502 - proxy не получил корректный ответ от upstream или не смог нормально с ним взаимодействовать. 504 - proxy ждал upstream, но истек timeout.

Вопрос: что важнее увеличить при загрузке файлов: `upload_max_filesize` или `client_max_body_size`?

Ответ: оба и еще `post_max_size`, плюс лимиты CDN/LB и app validation. Самое маленькое ограничение в цепочке победит.

Вопрос: почему нельзя просто поставить большие timeouts?

Ответ: длинные timeouts держат соединения, воркеры, память и DB locks. При деградации они усиливают saturation. Лучше иметь короткие осознанные budgets, async jobs и backpressure.

## Pitfalls для интервью

- Говорить, что DNS failover мгновенный из-за низкого TTL.
- Считать CORS механизмом защиты backend от всех клиентов.
- Доверять `X-Forwarded-For` без trusted proxy config.
- Путать `no-cache` и `no-store`.
- Retry всех `POST` без idempotency key.
- Лечить 504 только увеличением timeout.
- Не учитывать, что 499 может быть вызван client/CDN/LB timeout.
- Игнорировать `pm.max_children` и memory sizing PHP-FPM.
- Включать sticky sessions вместо выноса session state.
- Забывать `Vary` при кешировании CORS или compressed responses.
- Не различать client-side HTTP/2 и backend FastCGI.

## Self-check

Ответьте без подсказок:

1. Что происходит между вводом URL и попаданием запроса в PHP-контроллер?
2. Чем `proxy_read_timeout` отличается от `proxy_connect_timeout`?
3. Почему 504 может не остановить PHP-код?
4. Как отличить saturation PHP-FPM от медленного SQL?
5. Почему `Access-Control-Allow-Origin: *` несовместим с credentials?
6. Какие headers нужны для безопасной session cookie?
7. Почему `Vary: Origin` важен при dynamic CORS?
8. Чем `ETag` отличается от `Cache-Control`?
9. Когда retry безопасен, а когда опасен?
10. Что означает `upstream sent too big header`?
11. Какие лимиты проверять при `413 Payload Too Large`?
12. Почему load balancing по IP может работать плохо?
13. Что смотреть первым при 502 от Nginx к PHP-FPM?
14. Почему большие cookies ухудшают производительность?
15. Чем HTTP/3 отличается от HTTP/2 на транспортном уровне?

## Mini-practice

Задача 1: API иногда возвращает 504 через 60 секунд, но в логах приложения видно успешное завершение через 75 секунд.

Ожидаемый разбор: Nginx/LB timeout меньше времени выполнения app. Нужно проверить `proxy_read_timeout`/`fastcgi_read_timeout`, PHP-FPM `request_terminate_timeout`, `max_execution_time`, client timeout. Исправление не должно быть только увеличением timeout: вынести long-running работу в queue, вернуть `202 Accepted`, добавить idempotency и статус операции.

Задача 2: После релиза выросли 502, в Nginx error log есть `connect() to unix:/run/php/php-fpm.sock failed (11: Resource temporarily unavailable)`.

Ожидаемый разбор: FPM socket/backlog/workers перегружены. Проверить `pm.max_children reached`, CPU/memory, slowlog, DB latency, backlog. Возможно, новый код дольше держит воркеры. Нужны profiling, увеличение capacity только после memory sizing, rollback или hotfix.

Задача 3: Пользователи за корпоративным NAT массово получают 429.

Ожидаемый разбор: rate limit по IP слишком грубый. Использовать user/API key/tenant key после authentication, IP оставить fallback. Проверить trusted proxy и реальный client IP.

Задача 4: Frontend не может отправить authenticated request, browser пишет CORS error.

Ожидаемый разбор: проверить preflight `OPTIONS`, `Access-Control-Allow-Origin` не `*`, `Access-Control-Allow-Credentials: true`, allowed headers, allowed methods, cookies `SameSite=None; Secure` для cross-site, `Vary: Origin`.

Задача 5: SSE endpoint отдает события пачкой раз в минуту, хотя приложение flush делает сразу.

Ожидаемый разбор: вероятно включен Nginx buffering или buffering на CDN. Отключить для endpoint через `X-Accel-Buffering: no`, `proxy_buffering off`/`fastcgi_buffering off`, проверить gzip и CDN.

## Checklist перед интервью

- Умею нарисовать путь запроса: DNS -> TCP -> TLS -> HTTP -> Nginx -> PHP-FPM -> app -> DB/cache.
- Знаю различия HTTP/1.1, HTTP/2, HTTP/3.
- Могу объяснить keep-alive и его риски.
- Понимаю `X-Forwarded-*` и trusted proxies.
- Различаю `Cache-Control: no-cache` и `no-store`.
- Знаю cookie attributes: `HttpOnly`, `Secure`, `SameSite`, `Domain`, `Path`.
- Понимаю CORS и preflight.
- Могу объяснить reverse proxy, buffering и compression.
- Знаю базовые Nginx + PHP-FPM настройки.
- Понимаю upload limits на всех слоях.
- Могу построить timeout budget.
- Понимаю retries, idempotency key, backoff и jitter.
- Различаю 499, 502, 504 и знаю, где смотреть логи.
- Могу объяснить rate limiting и выбор ключа лимита.
- Говорю не только про config, но и про observability: logs, metrics, traces, slowlog.

## Ссылки

- [MDN: HTTP](https://developer.mozilla.org/en-US/docs/Web/HTTP)
- [MDN: HTTP caching](https://developer.mozilla.org/en-US/docs/Web/HTTP/Caching)
- [MDN: CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS)
- [MDN: Cookies](https://developer.mozilla.org/en-US/docs/Web/HTTP/Cookies)
- [MDN: HTTP response status codes](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status)
- [Nginx docs: Beginner's Guide](https://nginx.org/en/docs/beginners_guide.html)
- [Nginx docs: ngx_http_proxy_module](https://nginx.org/en/docs/http/ngx_http_proxy_module.html)
- [Nginx docs: ngx_http_fastcgi_module](https://nginx.org/en/docs/http/ngx_http_fastcgi_module.html)
- [Nginx docs: ngx_http_upstream_module](https://nginx.org/en/docs/http/ngx_http_upstream_module.html)
- [Nginx docs: ngx_http_limit_req_module](https://nginx.org/en/docs/http/ngx_http_limit_req_module.html)
- [PHP manual: FastCGI Process Manager](https://www.php.net/manual/en/install.fpm.php)
- [RFC 9110: HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110)
- [RFC 9111: HTTP Caching](https://www.rfc-editor.org/rfc/rfc9111)
- [RFC 9112: HTTP/1.1](https://www.rfc-editor.org/rfc/rfc9112)
- [RFC 9113: HTTP/2](https://www.rfc-editor.org/rfc/rfc9113)
- [RFC 9000: QUIC](https://www.rfc-editor.org/rfc/rfc9000)
- [RFC 8446: TLS 1.3](https://www.rfc-editor.org/rfc/rfc8446)
- [RFC 1034: Domain Names - Concepts and Facilities](https://www.rfc-editor.org/rfc/rfc1034)
- [RFC 1035: Domain Names - Implementation and Specification](https://www.rfc-editor.org/rfc/rfc1035)
