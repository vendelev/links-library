# HTTP, DNS, TLS, Nginx, reverse proxy

Конспект о пути запроса от клиента до PHP-кода, сетевых слоях, reverse proxy, HTTP semantics, Nginx/FastCGI и типичных 499/502/504.

## Карта запроса

Типовой путь:

1. Клиент получает IP через DNS.
2. Клиент открывает TCP-соединение или использует keep-alive.
3. Для HTTPS выполняется TLS-handshake.
4. Клиент отправляет HTTP-запрос.
5. Reverse proxy, например Nginx, принимает запрос.
6. Nginx выбирает upstream: приложение, балансировщик, PHP-FPM через FastCGI, CDN или другой сервис.
7. PHP-FPM исполняет PHP-код, обращается к БД, кешу, очередям, внешним API.
8. Ответ проходит обратно через Nginx и сеть к клиенту.

Senior-ответ:

> Важно уметь разложить latency по этапам: DNS, connect, TLS, TTFB, upstream processing, download. Без этого диагностика 502/504/499 превращается в угадывание.

## DNS basics

DNS преобразует имя домена в IP-адрес и другие записи.

Основные записи:

- `A`: IPv4-адрес.
- `AAAA`: IPv6-адрес.
- `CNAME`: алиас на другое DNS-имя.
- `MX`: почтовые серверы.
- `TXT`: SPF/DKIM/DMARC, верификации, произвольные данные.
- `NS`: authoritative name servers зоны.
- `CAA`: какие CA могут выпускать TLS-сертификаты для домена.

Ключевые понятия:

- Recursive resolver: резолвер провайдера, Google DNS, Cloudflare DNS, корпоративный DNS.
- Authoritative DNS: источник истины для зоны.
- TTL: время кеширования записи.
- Negative caching: кеширование отсутствия записи, например `NXDOMAIN`.
- Split-horizon DNS: разные ответы для внутренних и внешних клиентов.

Pitfalls:

- высокий TTL замедляет миграции и аварийное переключение;
- низкий TTL не гарантирует мгновенный failover;
- `CNAME` на apex-домене обычно нельзя использовать в классическом DNS, нужны `ALIAS`/`ANAME` у провайдера;
- IPv6 ломает доступ, если `AAAA` указывает на неработающий endpoint;
- DNS round-robin не является полноценным load balancing, потому что нет health checks на стороне DNS-клиента.

Senior-ответ:

> DNS не знает о состоянии приложения, а TTL не является SLA на переключение. Для надежного failover нужны health checks, балансировщики, anycast/CDN или управляемый DNS с проверками.

## TCP и TLS

TCP дает надежную доставку байтов поверх IP.

Свойства TCP:

- 3-way handshake: `SYN`, `SYN-ACK`, `ACK`;
- порядок доставки и retransmission;
- flow control и congestion control;
- состояние соединения на клиенте, сервере и промежуточных устройствах;
- закрытие через `FIN` или аварийный `RST`.

TLS обеспечивает шифрование, аутентификацию сервера и целостность данных.

TLS handshake включает:

- согласование версии TLS и cipher suite;
- проверку сертификата и цепочки доверия;
- обмен ключами;
- после handshake HTTP идет внутри зашифрованного канала.

TLS 1.3 обычно быстрее TLS 1.2: меньше round trips, современные cipher suites, session resumption.

Pitfalls:

- просроченный сертификат или неполная цепочка ломают клиентов;
- неправильный SNI приводит к выдаче не того сертификата;
- старые клиенты могут не поддерживать современные настройки TLS;
- OCSP/CRL и проверка сертификатов могут добавлять latency;
- TLS termination на reverse proxy означает, что дальше до upstream может идти HTTP или отдельный TLS.

Senior-ответ:

> HTTPS-ошибка не всегда про приложение. Нужно проверять DNS, SNI, цепочку сертификата, ALPN, TLS-версию, cipher suite, termination point и proxy-to-upstream настройки.

## HTTP/1.1, HTTP/2, HTTP/3

HTTP/1.1:

- текстовый протокол;
- один запрос активно обрабатывается на соединении в конкретный момент, pipelining почти не используется;
- keep-alive переиспользует TCP/TLS-соединение;
- head-of-line blocking на уровне соединения.

HTTP/2:

- бинарный протокол;
- мультиплексирование нескольких stream поверх одного TCP-соединения;
- HPACK-сжатие заголовков;
- server push устарел и практически не рекомендуется;
- TCP head-of-line blocking все еще возможен при потере пакетов.

HTTP/3:

- работает поверх QUIC/UDP;
- TLS 1.3 встроен в QUIC;
- меньше проблем с TCP head-of-line blocking между stream;
- лучше переносит смену сети, например Wi-Fi на LTE;
- требует поддержки на клиенте, edge/proxy и сети.

Pitfalls:

- HTTP/2 не ускоряет медленный backend сам по себе;
- один HTTP/2 connection может концентрировать много запросов, поэтому limits важны;
- между клиентом и Nginx может быть HTTP/2, а между Nginx и PHP-FPM - FastCGI;
- HTTP/3 может блокироваться сетями, где UDP ограничен.

## Keep-alive

Плюсы:

- меньше TCP/TLS handshakes;
- ниже latency;
- меньше CPU на TLS;
- выше throughput.

Минусы и риски:

- idle-соединения занимают память и file descriptors;
- слишком длинный keep-alive держит ресурсы без пользы;
- несовпадение timeout между proxy, load balancer и app вызывает обрывы;
- connection reuse может проявлять stale upstream connections.

Nginx-настройки:

- `keepalive_timeout`: сколько держать idle client connection;
- `keepalive_requests`: сколько запросов разрешить на одном соединении;
- `upstream keepalive`: пул keep-alive соединений к HTTP upstream.

## HTTP headers

Частые request headers:

- `Host`;
- `User-Agent`;
- `Accept`, `Accept-Encoding`, `Accept-Language`;
- `Authorization`;
- `Cookie`;
- `Content-Type`;
- `Content-Length`;
- `X-Forwarded-For`;
- `X-Forwarded-Proto`;
- `Forwarded`.

Частые response headers:

- `Content-Type`;
- `Content-Length` или chunked transfer;
- `Cache-Control`, `ETag`, `Last-Modified`;
- `Set-Cookie`;
- `Location`;
- `Vary`;
- `Strict-Transport-Security`;
- `Content-Security-Policy`.

Pitfalls:

- нельзя доверять `X-Forwarded-For` от клиента без trusted proxies;
- отсутствие `Vary: Origin` при CORS и кешировании может отдавать неверные ответы разным origin;
- неверный `Content-Length` вызывает hanging requests или protocol errors;
- headers могут быть ограничены по размеру в Nginx, CDN, LB и PHP.

Senior-ответ:

> Headers являются частью security boundary. Особенно важно корректно обрабатывать forwarded headers, cookies, auth headers и cache headers.

## HTTP caching headers

Основные заголовки:

- `Cache-Control`: главная политика кеширования;
- `Expires`: устаревший absolute expiry;
- `ETag`: validator версии ресурса;
- `Last-Modified`: validator по времени изменения;
- `Vary`: ключ варианта кеша;
- `Age`: сколько ответ уже провел в shared cache.

Директивы `Cache-Control`:

- `no-store`: не сохранять нигде;
- `no-cache`: можно сохранить, но перед использованием нужно revalidate;
- `private`: только private cache, например браузер;
- `public`: можно shared cache;
- `max-age=SECONDS`: свежесть для клиента;
- `s-maxage=SECONDS`: свежесть для shared cache;
- `must-revalidate`: нельзя использовать stale без проверки;
- `immutable`: ресурс не изменится во время freshness lifetime.

Стратегии:

- static assets: `Cache-Control: public, max-age=31536000, immutable` с fingerprint в имени файла;
- HTML/API user-specific: часто `private, no-store` или короткий `max-age`;
- public API lists: `ETag`/`Last-Modified` плюс аккуратный `Vary`.

Pitfalls:

- кеширование персональных данных через shared cache;
- отсутствие `Vary: Authorization` или запрет кеша для auth-запросов;
- использование `no-cache` как полного запрета сохранения вместо `no-store`;
- кеш без invalidation/versioning-стратегии становится источником старых данных.

## Cookies

Cookie хранит небольшие данные на клиенте и отправляется на подходящие запросы.

Атрибуты:

- `HttpOnly`: недоступна из JavaScript;
- `Secure`: отправляется только по HTTPS;
- `SameSite=Lax|Strict|None`: контроль cross-site отправки;
- `Domain`: область доменов;
- `Path`: область путей;
- `Expires`/`Max-Age`: срок жизни.

Security practices:

- session cookie: `HttpOnly; Secure; SameSite=Lax` как базовая настройка;
- для third-party/cross-site cookie нужен `SameSite=None; Secure`;
- не хранить секреты в cookie без подписи/шифрования;
- минимизировать размер cookie, потому что она отправляется на каждый подходящий запрос.

Pitfalls:

- большие cookies увеличивают latency и могут превысить limits headers;
- неверный `Domain` может открыть cookie поддоменам;
- `SameSite=None` без `Secure` отклоняется современными браузерами;
- cookie-based auth требует защиты от CSRF.

## CORS

CORS ограничивает, какие browser origins могут читать ответы cross-origin запросов.

Основные заголовки:

- `Access-Control-Allow-Origin`;
- `Access-Control-Allow-Methods`;
- `Access-Control-Allow-Headers`;
- `Access-Control-Allow-Credentials`;
- `Access-Control-Max-Age`.

Preflight:

- браузер отправляет `OPTIONS` перед небезопасным cross-origin запросом;
- сервер должен ответить разрешенными методами и заголовками;
- preflight можно кешировать через `Access-Control-Max-Age`.

Pitfalls:

- `Access-Control-Allow-Origin: *` нельзя использовать вместе с credentials;
- CORS не защищает backend от server-to-server запросов;
- CORS - browser security mechanism, не authentication и не authorization;
- при dynamic origin нужен `Vary: Origin`;
- redirect во время preflight часто ломает запрос.

## Reverse proxy и load balancing

Reverse proxy принимает запросы от клиентов и проксирует их к backend.

Задачи:

- TLS termination;
- routing по host/path/header;
- load balancing;
- static files;
- compression;
- request/response buffering;
- rate limiting;
- access logs;
- header normalization;
- security headers.

Nginx часто стоит перед:

- PHP-FPM через FastCGI;
- HTTP backend, например Symfony/Laravel Octane, Swoole, RoadRunner, Node.js;
- API gateway;
- internal services.

Pitfalls:

- потеря исходного IP без `X-Forwarded-For` и trusted proxy config;
- неверный `X-Forwarded-Proto` ломает redirects, secure cookies и генерацию URL;
- конфликт timeout между CDN, LB, Nginx и app;
- proxy buffering может скрывать streaming и SSE.

Load balancing algorithms:

- round-robin;
- least connections;
- IP hash;
- hash by cookie/header/URI;
- weighted.

Health checks:

- passive: upstream помечается failed после ошибок;
- active: балансировщик сам проверяет endpoint;
- Nginx open source имеет passive checks, active checks доступны в Nginx Plus или внешними средствами.

Sticky sessions risks:

- сложнее масштабирование;
- деградация при падении узла;
- state лучше хранить во внешнем хранилище: Redis, DB, shared session storage.

Pitfalls:

- retry небезопасных методов может создать дублирующие операции;
- balancing по IP плохо работает за NAT и мобильными сетями;
- нет смысла балансировать на мертвые upstream без health checks;
- разные версии приложения за балансировщиком требуют совместимости схемы данных и API.

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

Для фронт-контроллера:

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

Nginx параметры:

- `fastcgi_pass`: socket или TCP endpoint FPM;
- `fastcgi_param SCRIPT_FILENAME`: путь до PHP-файла;
- `fastcgi_read_timeout`: ожидание ответа от FPM;
- `fastcgi_connect_timeout`: подключение к FPM;
- `fastcgi_send_timeout`: отправка запроса в FPM;
- `fastcgi_buffers`, `fastcgi_buffer_size`: буферизация ответа.

PHP-FPM параметры:

- `pm = dynamic|static|ondemand`;
- `pm.max_children`: максимум воркеров;
- `pm.max_requests`: перезапуск воркера после N запросов;
- `request_terminate_timeout`: жесткое ограничение времени PHP-запроса;
- `slowlog`: лог медленных запросов;
- `listen.backlog`: очередь подключений к FPM.

Pitfalls:

- неверный `SCRIPT_FILENAME` может привести к 404, 502 или security issue;
- `pm.max_children` слишком мал: очередь, 502/504, рост latency;
- `pm.max_children` слишком велик: memory exhaustion и swap;
- `request_terminate_timeout` должен быть согласован с Nginx timeout;
- `clear_env` влияет на доступность env-переменных.

Senior-ответ:

> Для PHP-FPM capacity planning считают память одного worker под реальной нагрузкой, затем выбирают `pm.max_children`, чтобы не уйти в swap. CPU, DB pool и external calls тоже ограничивают throughput.

## Buffering, compression, upload limits

Nginx может буферизовать request body и response body.

Request buffering:

- Nginx читает тело запроса от клиента до передачи upstream;
- защищает backend от slow clients;
- может писать большие тела во временные файлы.

Response buffering:

- Nginx читает ответ upstream в буферы и отдает клиенту своим темпом;
- освобождает upstream быстрее при медленном клиенте;
- может мешать streaming, SSE и long polling.

Директивы:

- `proxy_request_buffering` / `fastcgi_request_buffering`;
- `proxy_buffering` / `fastcgi_buffering`;
- `client_body_buffer_size`;
- `proxy_buffers`, `fastcgi_buffers`;
- `X-Accel-Buffering: no` для отключения buffering в отдельных ответах.

Compression:

- Gzip широко поддерживается и встроен в Nginx;
- Brotli часто дает лучшее сжатие text assets, но требует модуля Nginx или CDN;
- сжимать HTML, CSS, JS, JSON, SVG, XML;
- не сжимать JPEG, PNG, WebP, AVIF, MP4, ZIP, PDF с внутренним сжатием;
- учитывать `Accept-Encoding` и `Vary: Accept-Encoding`;
- dynamic compression под высокой нагрузкой может съедать CPU;
- для static assets лучше precompressed `.br`/`.gz` плюс fingerprint caching.

Upload limits:

- CDN/WAF;
- Load balancer;
- Nginx `client_max_body_size`;
- PHP `post_max_size`;
- PHP `upload_max_filesize`;
- application validation;
- disk/temp storage.

Симптомы:

- `413 Payload Too Large` от Nginx или CDN;
- пустой `$_POST`/`$_FILES`, если `post_max_size` превышен;
- 502/504 при долгой обработке upload;
- disk full из-за временных файлов.

Senior-ответ:

> Большие файлы лучше выгружать напрямую в object storage, а backend должен выдавать signed URL и обрабатывать metadata/asynchronous scanning.

## Status codes и rate limiting

Группы:

- `1xx`: informational;
- `2xx`: success;
- `3xx`: redirection;
- `4xx`: client error;
- `5xx`: server error.

Частые коды:

- `200 OK`, `201 Created`, `202 Accepted`, `204 No Content`;
- `301`, `302`, `303`, `307`, `308`;
- `400`, `401`, `403`, `404`, `409`, `412`, `413`, `415`, `422`, `429`;
- `500`, `502`, `503`, `504`;
- Nginx-specific `499`: клиент закрыл соединение до ответа.

Pitfalls:

- путать `401` и `403`;
- возвращать `500` на validation errors;
- использовать `200` с error body для ошибок API;
- не отдавать `Retry-After` при `429`/`503`.

Rate limiting защищает от abuse, spikes и случайных перегрузок.

Алгоритмы:

- fixed window;
- sliding window;
- token bucket;
- leaky bucket.

Где применять:

- CDN/WAF;
- Nginx для IP/path-level limits;
- application для user/account/API key limits;
- queue для background workloads.

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

- лимит по IP ломает пользователей за NAT;
- нужно учитывать trusted proxy, иначе лимит применяется к IP load balancer;
- без `Retry-After` клиенты не знают, когда повторить запрос;
- rate limiting не заменяет quotas и abuse detection.

## Troubleshooting 499, 502, 504

### 499 Client Closed Request

Что значит:

- клиент закрыл соединение до того, как Nginx отдал ответ;
- это может быть браузер, мобильное приложение, CDN, load balancer или timeout клиента.

Причины:

- backend отвечает дольше client timeout;
- пользователь отменил запрос;
- CDN/LB оборвал соединение раньше Nginx;
- мобильная сеть нестабильна;
- большой response download.

Что проверять:

- `$request_time`;
- `$upstream_response_time`;
- user agent, endpoint, размер ответа;
- timeouts CDN/LB/client;
- backend latency и slow logs.

### 502 Bad Gateway

Что значит:

- Nginx не смог получить корректный ответ от upstream.

Причины PHP-FPM:

- PHP-FPM не запущен;
- неверный socket/path/permissions;
- все FPM workers заняты, backlog переполнен;
- PHP process crashed или был убит OOM killer;
- неверный FastCGI response;
- слишком большой header от upstream.

Что проверять:

- Nginx error log: `connect() failed`, `upstream prematurely closed connection`, `recv() failed`, `upstream sent too big header`;
- статус PHP-FPM и pool logs;
- `pm.max_children reached`;
- OOM killer logs;
- socket permissions;
- deploy/restart события.

### 504 Gateway Timeout

Что значит:

- Nginx подключился к upstream, но не дождался ответа в пределах read timeout.

Причины:

- медленный SQL;
- lock contention;
- внешний API завис;
- очередь в PHP-FPM;
- deadlock или бесконечная операция;
- неверный timeout для long-running endpoint.

Что проверять:

- `$upstream_response_time` близок к `fastcgi_read_timeout`/`proxy_read_timeout`;
- PHP slowlog;
- DB slow query log;
- traces/APM;
- saturation PHP-FPM workers;
- external dependency latency.

Senior-ответ:

> 504 не лечится механическим увеличением timeout. Сначала нужно понять, где потрачено время: очередь FPM, PHP execution, SQL, network call или lock.

## Практический алгоритм диагностики

1. Проверить, воспроизводится ли проблема и на каком endpoint.
2. Посмотреть access log: status, `$request_time`, `$upstream_response_time`, bytes, user agent.
3. Посмотреть Nginx error log по request id или времени.
4. Проверить upstream health: PHP-FPM status, workers, backlog, CPU, memory, OOM.
5. Проверить application logs и slow logs.
6. Проверить DB/cache/external dependencies.
7. Сравнить timeouts по цепочке: client, CDN, LB, Nginx, PHP-FPM, app clients.
8. Сформулировать fix: capacity, query optimization, timeout budget, queue, caching, retry/idempotency, rate limit.

Nginx log variables:

- `$request_time`;
- `$upstream_response_time`;
- `$upstream_connect_time`;
- `$upstream_header_time`;
- `$status`;
- `$upstream_status`;
- `$request_length`;
- `$bytes_sent`;
- `$http_x_forwarded_for`.

## Типовые senior-вопросы

**Почему пользователь получил 504, а операция в базе выполнилась?**

> Proxy timeout истек раньше завершения PHP-кода или DB operation. Nginx закрыл клиентский запрос, но PHP/FPM мог продолжить выполнение. Нужны app-level timeout/cancellation, idempotency key, транзакции и корректный timeout budget.

**Почему после включения HTTP/2 backend не стал быстрее?**

> HTTP/2 уменьшает overhead соединений на участке client-proxy, но если latency в PHP, SQL или внешнем API, bottleneck не изменится.

**Почему `X-Forwarded-For` опасен?**

> Клиент может прислать этот header сам. Доверять ему можно только от trusted proxies, иначе можно обойти IP-based ACL, rate limit или audit.

**Когда отключать proxy buffering?**

> Для streaming/SSE/long polling, где клиент должен получать chunks сразу. Для обычных API buffering полезен, потому что защищает upstream от slow clients.

**Чем отличается 502 от 504?**

> 502 - proxy не получил корректный ответ от upstream или не смог с ним взаимодействовать. 504 - proxy ждал upstream, но истек timeout.

## Pitfalls

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

## Mini-practice

1. API возвращает 504 через 60 секунд, но приложение завершает работу через 75 секунд: найти mismatch Nginx/LB timeout и app runtime, вынести long-running работу в queue, вернуть `202 Accepted`, добавить idempotency и статус операции.
2. После релиза выросли 502, в error log `connect() to unix:/run/php/php-fpm.sock failed (11: Resource temporarily unavailable)`: проверить FPM socket/backlog/workers, `pm.max_children`, CPU/memory, slowlog, DB latency, rollback или hotfix.
3. Пользователи за корпоративным NAT получают 429: заменить IP key на user/API key/tenant key, IP оставить fallback, проверить trusted proxy.
4. Browser пишет CORS error: проверить preflight `OPTIONS`, `Access-Control-Allow-Origin` не `*`, credentials, allowed headers/methods, cookies `SameSite=None; Secure`, `Vary: Origin`.
5. SSE endpoint отдает события пачкой: проверить Nginx/CDN buffering, `X-Accel-Buffering: no`, `proxy_buffering off`/`fastcgi_buffering off`, gzip и CDN.

## Self-check

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
- [Nginx docs: request processing](https://nginx.org/en/docs/http/request_processing.html)
- [PHP manual: FastCGI Process Manager](https://www.php.net/manual/en/install.fpm.php)
- [RFC 9110: HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110)
- [RFC 9111: HTTP Caching](https://www.rfc-editor.org/rfc/rfc9111)
- [RFC 9112: HTTP/1.1](https://www.rfc-editor.org/rfc/rfc9112)
- [RFC 9113: HTTP/2](https://www.rfc-editor.org/rfc/rfc9113)
- [RFC 9000: QUIC](https://www.rfc-editor.org/rfc/rfc9000)
- [RFC 8446: TLS 1.3](https://www.rfc-editor.org/rfc/rfc8446)
- [RFC 1034: Domain Names - Concepts and Facilities](https://www.rfc-editor.org/rfc/rfc1034)
- [RFC 1035: Domain Names - Implementation and Specification](https://www.rfc-editor.org/rfc/rfc1035)
