# System Design Interview for Lead/Senior PHP Backend

<!-- markdownlint-disable MD013 -->

Конспект для system design interviews. Фокус: как вести дизайн-сессию, какие решения предлагать, какие компромиссы проговаривать и как не утонуть в деталях реализации.

## Цель System Design Interview

Цель не в том, чтобы сразу нарисовать идеальную архитектуру. Цель - показать инженерное мышление:

- уточнение требований;
- приоритизация;
- декомпозиция;
- trade-offs;
- понимание отказов;
- операционная эксплуатация.

Рабочий порядок ответа:

1. Уточнить задачу и границы системы.
2. Сформулировать functional requirements.
3. Сформулировать non-functional requirements.
4. Сделать capacity estimates.
5. Предложить high-level architecture.
6. Спроектировать API и основные потоки данных.
7. Выбрать data model и storage.
8. Разобрать масштабирование, caching, queues, consistency, fault tolerance.
9. Обсудить observability, security, rollout, disaster recovery.
10. Явно назвать trade-offs и alternatives.

Фраза-шаблон:

> Я сначала уточню требования и целевые нагрузки, затем предложу базовую архитектуру, после этого пройдусь по узким местам: хранение, кеши, очереди, консистентность, отказоустойчивость и эксплуатация.

## Requirements Clarification

На senior-level нельзя начинать с технологий. Сначала выясняются границы продукта.

Вопросы:

- Кто пользователи: клиенты, админы, внешние сервисы, партнеры?
- Какие основные сценарии обязательны для MVP?
- Что не входит в систему?
- Нужны ли real-time обновления?
- Какой SLA/SLO ожидается?
- Где важнее consistency, а где допустима eventual consistency?
- Есть ли compliance: GDPR, PCI DSS, персональные данные?
- Нужна ли multi-region архитектура?
- Какие ограничения по бюджету, latency, команде, срокам?

Пример:

> Для notification service я уточню каналы: email, SMS, push, webhook; требования к доставке; допустимые задержки; нужна ли дедупликация; какие провайдеры; как обрабатывать bounce и retries.

## Functional Requirements

Functional requirements описывают, что система делает.

Пример URL shortener:

- создать короткую ссылку;
- перенаправить по короткому коду;
- поддержать custom alias;
- учитывать TTL ссылки;
- собирать клики для аналитики;
- защититься от spam/abuse.

Пример order processing:

- создать заказ;
- зарезервировать товар;
- провести оплату;
- обновить статус заказа;
- отправить уведомления;
- поддержать отмену и возврат.

## Non-Functional Requirements

Non-functional requirements описывают качество системы.

Основные категории:

- Latency: например, p95 чтения меньше 100 ms.
- Throughput: запросов/событий в секунду.
- Availability: например, 99.9% или 99.99%.
- Consistency: strong, read-your-writes, eventual.
- Durability: потеря данных недопустима или допустима частичная потеря аналитики.
- Scalability: горизонтальное масштабирование backend workers и storage.
- Security: auth, authorization, encryption, audit.
- Operability: monitoring, logs, traces, alerts, runbooks.
- Cost: где можно использовать managed services, где нельзя.

Старший ответ проговаривает приоритеты:

> Для редиректа URL shortener важны availability и low latency. Для аналитики кликов допустима eventual consistency, поэтому можно писать события асинхронно через queue или stream.

## Capacity Estimates

Оценки нужны не для точной математики, а для выбора порядка архитектуры.

Что считать:

- DAU/MAU;
- read/write ratio;
- RPS/QPS;
- размер записи;
- рост данных в день/месяц/год;
- пиковые нагрузки;
- количество сообщений в очередях;
- retention requirements.

Шаблон:

```text
Users: 10M MAU, 1M DAU
Reads: 100M/day ~= 1150 rps average, peak x5 ~= 5750 rps
Writes: 1M/day ~= 12 rps average, peak x10 ~= 120 rps
Record size: 1 KB
Storage/year: 1M * 1 KB * 365 ~= 365 GB before indexes/replication
```

Практический вывод:

- если writes низкие, reads высокие - cache, read replicas, CDN;
- если events много, а real-time точность не критична - queue/stream и batch aggregation;
- если storage быстро растет - partitioning, TTL, cold storage, retention policy.

## High-Level Architecture

Типовой backend layout для PHP:

```text
Client / Mobile / Partner API
        |
CDN / WAF / Load Balancer
        |
Laravel/Symfony API services
        |
PostgreSQL / Redis / Queue / Object Storage
        |
Workers / Consumers / Analytics pipeline
```

Для Laravel/Symfony:

- API service: REST/OpenAPI, auth, validation, orchestration.
- Domain services: бизнес-логика без привязки к HTTP.
- Repositories/DAO: доступ к PostgreSQL/ClickHouse/Redis.
- Queue jobs/consumers: Laravel Queue, Symfony Messenger.
- Scheduler: Laravel Scheduler, Symfony Console cron.
- Outbox publisher: надежная публикация событий после транзакции.

## API Design

Для интервью обычно достаточно REST/OpenAPI, если не требуется GraphQL/gRPC.

Принципы:

- версионирование: `/api/v1/...` или header versioning;
- idempotency для write operations;
- cursor pagination для больших списков;
- явные ошибки: code, message, details, trace_id;
- OpenAPI schema как контракт;
- backward-compatible изменения.

Пример order API:

```http
POST /api/v1/orders
Idempotency-Key: 6f3c...
Content-Type: application/json

{
  "user_id": "u_123",
  "items": [
    {"sku": "book-1", "qty": 2}
  ],
  "payment_method_id": "pm_456"
}
```

Ответ:

```json
{
  "id": "ord_123",
  "status": "pending_payment",
  "created_at": "2026-06-29T10:00:00Z"
}
```

Ошибка:

```json
{
  "error": {
    "code": "INSUFFICIENT_STOCK",
    "message": "Not enough stock for sku book-1",
    "trace_id": "abc123"
  }
}
```

## Data Model

Начинать нужно с сущностей и инвариантов, не с таблиц.

Что определить:

- основные entities;
- relationships;
- immutable events vs mutable state;
- уникальные ключи;
- индексы под query patterns;
- retention и архивирование.

Пример order processing в PostgreSQL:

```sql
orders(id, user_id, status, total_amount, currency, created_at, updated_at)
order_items(id, order_id, sku, qty, unit_price)
payments(id, order_id, provider, status, idempotency_key, created_at)
inventory_reservations(id, order_id, sku, qty, status, expires_at)
outbox_events(id, aggregate_type, aggregate_id, event_type, payload, published_at)
```

Индексы:

- `orders(user_id, created_at desc)` для истории заказов;
- `payments(idempotency_key)` unique для защиты от повторной оплаты;
- `outbox_events(published_at, id)` для publisher worker.

## Storage Choice

Выбор хранилища должен идти от access patterns.

### PostgreSQL

Подходит для:

- transactions;
- constraints;
- relational model;
- заказы, пользователи, платежи, настройки;
- default choice для PHP backend.

### Redis

Подходит для:

- cache;
- rate limiting;
- distributed locks с осторожностью;
- ephemeral state;
- sorted sets для leaderboard/feed ranking.

Redis Streams можно использовать для простых event pipelines, но для надежной очереди часто выбирают RabbitMQ/SQS/Kafka.

### ClickHouse

Подходит для:

- analytics;
- events;
- aggregates;
- time-series;
- OLAP-запросов по большим объемам.

Не основной source of truth для транзакционных данных.

### RabbitMQ/SQS

Подходит для:

- async tasks;
- retries;
- delayed processing;
- fanout;
- provider integrations.

RabbitMQ удобен для routing patterns. SQS хорош как managed queue с at-least-once delivery.

### Object Storage

Подходит для файлов, медиа, exports. В БД хранить metadata, не бинарные файлы.

## Caching

Cache нужен для latency, throughput и защиты БД, но добавляет invalidation complexity.

Стратегии:

- Cache-aside: приложение сначала смотрит Redis, потом БД.
- Write-through: запись идет через cache и storage.
- Write-behind: быстро, но есть риск потери данных.
- Read-through: cache сам загружает данные.

Что кешировать:

- часто читаемые профили, настройки, reference data;
- результаты дорогих запросов;
- feed pages с коротким TTL;
- rate limit counters.

Проблемы:

- Cache stampede: lock, jitter TTL, stale-while-revalidate.
- Invalidation: TTL, event-based invalidation, versioned keys.
- Hot keys: local in-memory cache, replication, key splitting.

Пример ключей:

```text
user:{id}:profile:v3
feed:{user_id}:cursor:{cursor_hash}
rate_limit:{user_id}:{minute}
```

## Queues and Async Processing

Очереди отделяют быстрый synchronous path от тяжелой работы.

Подходящие задачи:

- email/SMS/push;
- thumbnails;
- search indexing;
- analytics recalculation;
- external integrations;
- retry payment callback handling.

PHP примеры:

- Laravel Queue jobs + Horizon для Redis/RabbitMQ/SQS;
- Symfony Messenger + transports RabbitMQ/SQS/Doctrine/Redis;
- consumers как отдельные deployment units.

Важные свойства:

- at-least-once delivery означает, что consumer обязан быть idempotent;
- Dead Letter Queue для сообщений, которые не обработались;
- backoff retries для временных ошибок;
- SQS visibility timeout должен быть больше времени обработки;
- poison messages не должны блокировать очередь.

## Consistency

Strong consistency нужна не везде.

Где нужна strong consistency:

- оплата;
- inventory reservation;
- balances and ledger;
- уникальность custom alias;
- права доступа.

Где допустима eventual consistency:

- click analytics;
- view counters;
- feeds and recommendations;
- notifications;
- search indexes.

Паттерны:

- database transaction для локальной атомарности;
- outbox pattern для надежной публикации событий;
- saga для распределенных бизнес-процессов;
- idempotency keys для повторяемых запросов;
- optimistic locking через `version` или `updated_at`.

Trade-off:

> Для marketplace analytics я не буду писать каждый event синхронно в PostgreSQL. API отправит event в queue/stream, consumer батчами пишет в ClickHouse. Это снижает latency и нагрузку, но аналитика обновляется с задержкой.

## Availability

Availability достигается не одной технологией, а устранением single points of failure.

Практики:

- несколько stateless API instances;
- load balancer health checks;
- PostgreSQL primary + replicas + backups;
- Redis Sentinel/Cluster или managed Redis;
- queues с HA или managed SQS;
- graceful degradation;
- timeouts на внешние вызовы;
- circuit breakers для нестабильных dependencies.

Различать:

- High availability: система продолжает работать при отказах.
- Durability: данные не теряются.
- Fault tolerance: отказ части системы не валит все.

## Scalability

Сначала масштабировать stateless слой, потом stateful.

Подходы:

- horizontal scaling PHP-FPM/API containers;
- read replicas для read-heavy PostgreSQL;
- Redis cache для hot reads;
- queue workers autoscaling по queue depth;
- partitioning больших таблиц;
- sharding только когда один узел/кластер не справляется;
- CDN для media/static/downloads.

PHP details:

- PHP-FPM workers ограничены CPU/RAM; считать max children;
- OPcache обязателен в production;
- долгие задачи не выполнять в HTTP request lifecycle;
- Composer autoload optimized в production.

## Partitioning and Sharding

Partitioning - деление таблицы внутри одной логической БД. Sharding - деление данных по нескольким БД/кластерам.

Partitioning PostgreSQL:

- по времени: events, logs, analytics, audit;
- по tenant_id: multi-tenant SaaS;
- по hash user_id/order_id: равномерное распределение.

Плюсы partitioning:

- быстрое удаление старых данных через drop partition;
- partition pruning;
- уменьшение индексов на partition.

Sharding использовать, когда vertical scaling, indexes, caching, replicas и partitioning недостаточны.

Хорошие shard keys:

- `user_id` для user-centric systems;
- `tenant_id` для B2B SaaS;
- hash от id для равномерного распределения.

Плохие shard keys:

- timestamp, если нагрузка идет только в свежий диапазон;
- country/region, если распределение сильно неравномерное.

## Replication

Replication повышает availability и read scalability, но добавляет lag.

Типы:

- synchronous replication: выше consistency, ниже latency/availability;
- asynchronous replication: выше performance, возможен replication lag;
- logical replication: selective replication, CDC scenarios.

Pitfall:

> После записи пользователь может не увидеть свои данные, если чтение ушло на lagging replica. Для read-your-writes можно читать с primary короткое время после write или использовать session stickiness.

## Load Balancing

Load balancer распределяет трафик и изолирует отказавшие instances.

Уровни:

- L4: TCP/UDP, быстро, меньше контекста.
- L7: HTTP-aware routing, headers, paths, TLS termination.

Алгоритмы:

- round robin;
- least connections;
- weighted routing;
- consistent hashing для sticky workloads.

Для PHP API обычно:

```text
Nginx/ALB/Ingress -> PHP-FPM/API containers -> services/storage
```

Важно:

- stateless sessions, хранить session в Redis/DB/JWT при необходимости;
- health checks проверяют readiness, а не только process alive;
- graceful shutdown для in-flight requests.

## Rate Limiting

Rate limiting защищает систему от abuse и перегрузки.

Алгоритмы:

- Fixed window: просто, но burst на границе окна.
- Sliding window: точнее, дороже.
- Token bucket: позволяет burst в пределах bucket.
- Leaky bucket: сглаживает поток.

Где применять:

- per IP для публичных endpoints;
- per user/API key для authenticated API;
- per route для дорогих операций;
- per tenant для B2B fairness.

Redis пример:

```text
INCR rate:{api_key}:{minute}
EXPIRE rate:{api_key}:{minute} 120
```

Для distributed rate limiting increment+expire должны быть атомарными через Lua script или готовый middleware, иначе возможны race conditions.

## Idempotency

Idempotency означает, что повтор одного и того же запроса не приводит к повторному side effect.

Где обязательно:

- создание платежа;
- создание заказа;
- webhook callbacks;
- consumers очередей;
- external provider integrations.

Реализация high-level:

- клиент отправляет `Idempotency-Key`;
- сервер хранит key, request hash, response, status;
- unique constraint на key + actor/scope;
- повтор возвращает тот же результат;
- если request hash отличается - вернуть conflict.

Таблица:

```sql
idempotency_keys(key, actor_id, request_hash, response_body, status_code, expires_at, created_at)
```

Подробная реализация - в `05-concurrency-reliability.md`.

## Retries

Retries помогают при transient errors, но могут усилить аварию.

Правила:

- retry только для временных ошибок: timeout, 429, 503, network reset;
- не retry без idempotency;
- exponential backoff + jitter;
- ограничить max attempts;
- после лимита - DLQ/manual review;
- не retry validation/business errors.

Пример:

```text
attempt 1: immediately
attempt 2: 5s + jitter
attempt 3: 30s + jitter
attempt 4: 5m + jitter
then DLQ
```

## Circuit Breakers

Circuit breaker защищает систему от деградирующей зависимости.

Состояния:

- Closed: запросы проходят.
- Open: запросы быстро отклоняются или fallback.
- Half-open: пробные запросы для восстановления.

Где полезно:

- payment provider;
- SMS/email provider;
- external partner API;
- search/analytics dependency.

Fallback examples:

- вернуть cached data;
- поставить задачу в очередь;
- показать degraded response;
- переключиться на второго провайдера.

## Observability

Observability отвечает на вопрос: что происходит в production и почему.

Три базовых сигнала:

- Logs: structured JSON logs с `trace_id`, `user_id`, `order_id`.
- Metrics: RED/USE metrics, latency percentiles, error rate, saturation.
- Traces: distributed tracing через HTTP, queue, DB calls.

Что мониторить:

- API p50/p95/p99 latency;
- 4xx/5xx rate;
- queue depth и oldest message age;
- worker success/failure rate;
- PostgreSQL connections, locks, slow queries, replication lag;
- Redis memory, evictions, hit ratio;
- RabbitMQ ready/unacked messages, consumer count;
- ClickHouse insert latency, failed inserts, disk usage.

Alerts должны быть actionable:

- `p95 latency > SLO 10 минут`;
- `queue oldest message age > 5 минут`;
- `payment error rate > 2%`;
- `PostgreSQL replication lag > 30 секунд`.

## Security Basics

Минимальный набор для backend system design:

- TLS everywhere;
- authentication: session/JWT/OAuth2/OIDC по контексту;
- authorization: RBAC/ABAC, tenant isolation;
- input validation и output encoding;
- SQL injection защита через prepared statements/ORM query builder;
- secrets в secret manager, не в env dumps/logs;
- encryption at rest для sensitive data;
- audit log для критичных действий;
- rate limiting и WAF для публичных endpoints;
- principle of least privilege для DB/users/service accounts;
- PII minimization и retention policies.

PHP/Laravel/Symfony:

- не логировать токены, пароли, payment data;
- CSRF для browser forms;
- Secure cookies: HttpOnly, Secure, SameSite;
- password hashing: Argon2id/bcrypt через framework facilities;
- проверять serialization/deserialization risks.

## Deployment and Rollout

Хороший senior ответ включает безопасную доставку изменений.

Практики:

- blue/green deployment;
- canary release;
- rolling deployment;
- feature flags;
- backward-compatible DB migrations;
- expand/contract migration pattern;
- automated rollback;
- smoke tests после deploy.

DB migration порядок:

1. Add nullable column/new table.
2. Deploy code that writes both old and new if needed.
3. Backfill data.
4. Switch reads to new field.
5. Stop writing old field.
6. Drop old field later.

Для PHP:

- очистка/прогрев config cache, route cache, OPcache;
- совместимость старых workers с новыми message schemas;
- graceful restart consumers.

## Disaster Recovery

Disaster recovery - план восстановления после серьезного отказа.

Понятия:

- RPO: сколько данных можно потерять.
- RTO: как быстро нужно восстановиться.

Практики:

- regular backups;
- проверка restore, а не только наличие backup;
- PITR для PostgreSQL;
- cross-region backup для критичных данных;
- runbooks для failover;
- game days/chaos testing для зрелых команд;
- DLQ replay procedures.

Пример:

> Для order/payment системы RPO должен быть близок к нулю, RTO может быть 15-60 минут в зависимости от бизнеса. Для click analytics RPO может быть несколько минут, а восстановление возможно через replay raw events.

## Trade-Offs

Частые trade-offs:

- strong consistency vs availability/latency;
- synchronous API vs async processing;
- PostgreSQL simplicity vs specialized storage;
- cache performance vs invalidation complexity;
- sharding scalability vs operational complexity;
- managed services vs vendor lock-in/cost;
- microservices autonomy vs distributed complexity;
- real-time analytics vs batch simplicity;
- exactly-once illusion vs practical idempotent at-least-once processing.

Фраза-шаблон:

> Я бы начал с PostgreSQL + Redis + queue, потому что это покрывает текущий масштаб и проще в эксплуатации. ClickHouse добавил бы для аналитики при росте event volume. Sharding не вводил бы до доказанной необходимости, потому что он резко усложняет transactions, joins и rebalancing.

## Common Interview Tasks

### URL Shortener

Требования:

- create short URL;
- low-latency redirect;
- custom alias, TTL, analytics.

Архитектура:

- Laravel/Symfony API для create/manage;
- PostgreSQL для links metadata;
- Redis для hot code -> URL cache;
- Queue для click events;
- ClickHouse для analytics.

Ключевые моменты:

- unique constraint на code/custom alias;
- 301 vs 302 зависит от product requirements;
- генерация code: base62 sequence/random с collision handling;
- abuse protection и rate limiting;
- click analytics async.

Pitfalls:

- писать каждый click синхронно в PostgreSQL;
- не учитывать hot links;
- не иметь TTL/invalidation strategy.

### Feed

Требования:

- персональная лента;
- follow/unfollow;
- ranking, pagination, freshness.

Архитектура:

- PostgreSQL для users/posts/follows;
- Redis Sorted Sets для precomputed feed;
- Queue workers для fanout on write;
- fallback fanout on read для heavy users.

Trade-off:

- fanout on write: быстрые reads, дорогие writes;
- fanout on read: простые writes, дорогие reads;
- hybrid: celebrities/heavy users считаются при чтении.

Pitfalls:

- offset pagination на больших таблицах;
- не учитывать privacy changes;
- не иметь strategy для celebrity accounts.

### Marketplace Analytics

Требования:

- события просмотров, кликов, заказов;
- dashboards продавцов;
- агрегации по времени, категории, кампании.

Архитектура:

- API принимает events;
- queue/stream буферизует;
- consumers батчами пишут в ClickHouse;
- PostgreSQL хранит users/shops/products;
- Redis кеширует популярные dashboards.

Ключевые моменты:

- event schema versioning;
- deduplication через event_id;
- retention raw events и materialized views;
- eventual consistency приемлема.

Pitfalls:

- использовать PostgreSQL как OLAP при больших объемах;
- не версионировать event schema;
- не отделить raw events от aggregates.

### Notification Service

Требования:

- email/SMS/push/webhook;
- templates, preferences, unsubscribe;
- retries, provider failover, delivery status.

Архитектура:

- API для enqueue notification;
- PostgreSQL для templates/preferences/status;
- RabbitMQ/SQS для per-channel queues;
- workers на Symfony Messenger/Laravel Queue;
- Redis для rate limits и provider throttling.

Ключевые моменты:

- idempotency key на notification request;
- DLQ и retry policy;
- provider-specific rate limits;
- suppression list/unsubscribe compliance.

Pitfalls:

- синхронно отправлять SMS/email в API request;
- retry без deduplication;
- игнорировать provider quotas.

### File/Media Service

Требования:

- upload/download files;
- metadata, access control;
- thumbnails/transcoding;
- virus scanning.

Архитектура:

- API выдает pre-signed upload URL;
- Object Storage хранит binary;
- PostgreSQL хранит metadata/status/owner;
- Queue запускает scan/thumbnail/transcode jobs;
- CDN для downloads/public media.

Ключевые моменты:

- не проксировать большие файлы через PHP, если можно direct upload;
- multipart upload для больших файлов;
- signed URLs с TTL;
- async processing states: uploaded, scanning, ready, failed.

Pitfalls:

- хранить файлы в PostgreSQL без необходимости;
- не проверять content type/size;
- давать публичные permanent URLs для private files.

### Order Processing

Требования:

- создание заказа;
- inventory reservation;
- payment;
- status transitions;
- notifications/refunds.

Архитектура:

- Laravel/Symfony order API;
- PostgreSQL как source of truth;
- transaction для order + reservation;
- payment integration через idempotent calls;
- outbox events для notifications/analytics;
- queue workers для async side effects.

Ключевые моменты:

- state machine для order status;
- idempotency на create order/payment;
- saga для distributed flow;
- compensation для failed payment/reservation expiry;
- ledger-like модель для денег.

Pitfalls:

- держать DB transaction во время внешнего payment API call;
- не иметь unique idempotency key;
- смешивать аналитические события и transactional state.

## Answer Templates

### Начало ответа

```text
Сначала уточню scope. Я предполагаю, что нам нужен MVP с X, Y, Z. Не включаю A и B, если интервьюер не скажет обратное. Основные NFR: p95 latency, availability, durability и expected traffic. После этого предложу baseline architecture и отдельно разберу bottlenecks.
```

### Storage

```text
Для transactional source of truth я выберу PostgreSQL: нужны constraints, transactions и понятная модель данных. Redis использую как cache/rate limit/session store, но не как единственный durable storage. Для аналитики больших event volumes добавлю ClickHouse, потому что access pattern OLAP, а не OLTP.
```

### Queues

```text
Все side effects, которые не обязаны завершиться в HTTP request, вынесу в queue: notifications, analytics, media processing. Поскольку delivery обычно at-least-once, consumers делаю idempotent, добавляю retries with backoff и DLQ.
```

### Consistency

```text
В critical path заказа нужна strong consistency внутри PostgreSQL transaction. Межсервисные side effects публикую через outbox pattern. Для analytics и notifications принимаю eventual consistency, потому что это снижает latency и повышает resilience.
```

### Scaling

```text
Начну с stateless horizontal scaling API и workers, затем добавлю Redis cache и read replicas. Partitioning применю для больших time-series таблиц. Sharding оставлю как поздний шаг, потому что он сильно усложняет operational model.
```

## Common Pitfalls

- Сразу начинать с Kafka/Kubernetes/microservices без требований.
- Не считать хотя бы приблизительную нагрузку.
- Не различать OLTP и OLAP.
- Использовать cache как source of truth без объяснения durability.
- Игнорировать idempotency при retries.
- Обещать exactly-once без объяснения практической реализации.
- Держать транзакцию во время вызова внешнего API.
- Не обсуждать failure modes.
- Не говорить про observability и operations.
- Не учитывать schema evolution и backward compatibility.
- Не объяснять, почему sharding пока не нужен.
- Не учитывать GDPR/PII/security basics.

## Self-Check Questions

- Какие требования я уточнил перед архитектурой?
- Какой read/write ratio и порядок RPS?
- Где source of truth?
- Какие данные можно потерять, а какие нельзя?
- Где нужна strong consistency?
- Где acceptable eventual consistency?
- Какие endpoints idempotent?
- Какие операции уйдут в async processing?
- Как система ведет себя при падении Redis?
- Как система ведет себя при падении очереди?
- Что произойдет при лаге реплики PostgreSQL?
- Как обнаружить деградацию в production?
- Как безопасно выкатить новую схему БД?
- Как восстановиться из backup?
- Какие trade-offs я явно назвал?

## Mini-Practice

1. URL shortener за 20 минут: requirements, capacity, API/data model, caching/analytics, failure modes.
2. Notification service: API enqueue, tables, RabbitMQ или SQS, retries, DLQ, idempotency, provider failover.
3. Order processing: state machine, transaction boundary, external calls, outbox, compensations.
4. Marketplace analytics: event volume, ClickHouse schema, ingestion pipeline, retention, deduplication.

## Architecture Review Checklist

- Scope явно ограничен.
- Functional и non-functional requirements названы.
- Есть rough capacity estimates.
- API описан хотя бы для ключевых операций.
- Data model соответствует query patterns.
- Storage choices объяснены.
- Есть caching strategy и invalidation.
- Async processing вынесен из synchronous path.
- Idempotency покрывает writes и consumers.
- Retry policy безопасна.
- Consistency model объяснен по сценариям.
- Availability/failover рассмотрены.
- Scalability path идет от простого к сложному.
- Partitioning/sharding не вводятся преждевременно.
- Rate limiting и abuse protection есть.
- Observability покрывает logs, metrics, traces, alerts.
- Security basics не забыты.
- Deployment/rollback/migrations описаны.
- Disaster recovery имеет RPO/RTO.
- Trade-offs названы явно.

## Дополнительное чтение

- Martin Fowler, Architecture: <https://martinfowler.com/architecture/>
- Martin Fowler, Microservices: <https://martinfowler.com/articles/microservices.html>
- AWS Architecture Center: <https://aws.amazon.com/architecture/>
- AWS Well-Architected Framework: <https://aws.amazon.com/architecture/well-architected/>
- Google SRE Book: <https://sre.google/sre-book/table-of-contents/>
- Google SRE Workbook: <https://sre.google/workbook/table-of-contents/>
- Microsoft Azure Architecture Center: <https://learn.microsoft.com/en-us/azure/architecture/>
- Redis Documentation: <https://redis.io/docs/latest/>
- Redis Patterns: <https://redis.io/learn/howtos/solutions>
- RabbitMQ Documentation: <https://www.rabbitmq.com/docs>
- PostgreSQL Documentation: <https://www.postgresql.org/docs/>
- PostgreSQL Partitioning: <https://www.postgresql.org/docs/current/ddl-partitioning.html>
- ClickHouse Documentation: <https://clickhouse.com/docs>
- Amazon SQS Documentation: <https://docs.aws.amazon.com/sqs/>
- OpenAPI Specification: <https://spec.openapis.org/oas/latest.html>
- Designing Data-Intensive Applications, Martin Kleppmann: <https://dataintensive.net/>
- System Design Primer: <https://github.com/donnemartin/system-design-primer>
- Microsoft REST API Guidelines: <https://github.com/microsoft/api-guidelines>
- OWASP Application Security Verification Standard: <https://owasp.org/www-project-application-security-verification-standard/>
- OWASP API Security Top 10: <https://owasp.org/API-Security/>
