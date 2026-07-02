# System Design для Lead/Senior PHP Backend

Практический конспект для собеседований на Lead/Senior PHP backend: как вести дизайн-сессию, какие решения предлагать, какие компромиссы проговаривать и как не утонуть в деталях.

## Как проходить system design interview

Цель интервью не в том, чтобы сразу нарисовать идеальную архитектуру. Цель - показать инженерное мышление: уточнение требований, приоритизация, декомпозиция, trade-offs, понимание отказов и операционной эксплуатации.

Рабочий порядок ответа:

1. Уточнить задачу и границы системы.
2. Сформулировать functional requirements.
3. Сформулировать non-functional requirements.
4. Сделать capacity estimates.
5. Предложить high-level architecture.
6. Спроектировать API и основные потоки данных.
7. Выбрать data model и storage.
8. Разобрать масштабирование, кеширование, очереди, consistency, отказоустойчивость.
9. Обсудить observability, security, rollout, DR.
10. Явно назвать trade-offs и альтернативы.

Фраза-шаблон:

> Я сначала уточню требования и целевые нагрузки, затем предложу базовую архитектуру, после этого пройдусь по узким местам: хранение, кеши, очереди, консистентность, отказоустойчивость и эксплуатация.

## Уточнение требований

На старшем уровне важно не начинать с технологий. Сначала выясняются границы продукта.

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

Пример для URL shortener:

- Создать короткую ссылку.
- Перенаправить по короткому коду.
- Поддержать custom alias.
- Учитывать TTL ссылки.
- Собирать клики для аналитики.
- Защититься от spam/abuse.

Пример для order processing:

- Создать заказ.
- Зарезервировать товар.
- Провести оплату.
- Обновить статус заказа.
- Отправить уведомления.
- Поддержать отмену и возврат.

## Non-Functional Requirements

Non-functional requirements описывают качество системы.

Основные категории:

- Latency: например, p95 чтения меньше 100 мс.
- Throughput: запросов/событий в секунду.
- Availability: например, 99.9% или 99.99%.
- Consistency: strong, read-your-writes, eventual.
- Durability: потеря данных недопустима или допустима частичная потеря аналитики.
- Scalability: горизонтальное масштабирование backend workers и storage.
- Security: auth, authorization, encryption, audit.
- Operability: monitoring, logs, traces, alerts, runbooks.
- Cost: где можно использовать managed сервисы, где нельзя.

Старший ответ должен проговаривать приоритеты:

> Для редиректа URL shortener важны availability и low latency. Для аналитики кликов допустима eventual consistency, поэтому можно писать события асинхронно через очередь или stream.

## Capacity Estimates

Оценки нужны не для точной математики, а для выбора порядка архитектуры.

Что считать:

- DAU/MAU.
- Read/write ratio.
- RPS/QPS.
- Размер записи.
- Рост данных в день/месяц/год.
- Пиковые нагрузки.
- Количество сообщений в очередях.
- Требования к retention.

Шаблон:

```text
Users: 10M MAU, 1M DAU
Reads: 100M/day ~= 1150 rps average, peak x5 ~= 5750 rps
Writes: 1M/day ~= 12 rps average, peak x10 ~= 120 rps
Record size: 1 KB
Storage/year: 1M * 1 KB * 365 ~= 365 GB before indexes/replication
```

Практический вывод:

- Если writes низкие, reads высокие - кеш, read replicas, CDN.
- Если events много, а точность real-time не критична - очередь/stream и batch aggregation.
- Если storage растет быстро - partitioning, TTL, cold storage, retention policy.

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

Для интервью достаточно REST/OpenAPI, если не требуется GraphQL/gRPC.

Принципы:

- Версионирование: `/api/v1/...` или header versioning.
- Idempotency для write operations.
- Pagination: cursor pagination для больших списков.
- Явные ошибки: code, message, details, trace_id.
- OpenAPI schema как контракт.
- Backward compatible изменения.

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

Ошибки:

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

Начинайте с сущностей и инвариантов, не с таблиц.

Что определить:

- Основные entities.
- Relationships.
- Immutable events vs mutable state.
- Уникальные ключи.
- Индексы под query patterns.
- Retention и архивирование.

Пример order processing в PostgreSQL:

```sql
orders(id, user_id, status, total_amount, currency, created_at, updated_at)
order_items(id, order_id, sku, qty, unit_price)
payments(id, order_id, provider, status, idempotency_key, created_at)
inventory_reservations(id, order_id, sku, qty, status, expires_at)
outbox_events(id, aggregate_type, aggregate_id, event_type, payload, published_at)
```

Индексы:

- `orders(user_id, created_at desc)` для истории заказов.
- `payments(idempotency_key)` unique для защиты от повторной оплаты.
- `outbox_events(published_at, id)` для publisher worker.

## Storage Choice

Выбор хранилища должен идти от access patterns.

PostgreSQL:

- Транзакции, constraints, relational model.
- Заказы, пользователи, платежи, настройки.
- Хороший default choice для PHP backend.

Redis:

- Cache, rate limiting, distributed locks с осторожностью, ephemeral state.
- Sorted sets для leaderboard/feed ranking.
- Streams можно использовать для простых event pipelines, но для надежной очереди часто выбирают RabbitMQ/SQS/Kafka.

ClickHouse:

- Аналитика, events, агрегаты, time-series.
- Быстрые OLAP-запросы по большим объемам.
- Не основной источник истины для транзакционных данных.

RabbitMQ/SQS:

- Асинхронные задачи, retries, delayed processing, fanout.
- RabbitMQ удобен для routing patterns.
- SQS хорош как managed queue с at-least-once delivery.

Object Storage:

- S3-compatible storage для файлов, медиа, exports.
- В БД хранить metadata, не бинарные файлы.

## Caching

Кеш нужен для latency, throughput и защиты БД, но добавляет invalidation complexity.

Стратегии:

- Cache-aside: приложение сначала смотрит Redis, потом БД.
- Write-through: запись идет через кеш и storage.
- Write-behind: быстро, но риск потери данных.
- Read-through: кеш сам загружает данные.

Что кешировать:

- Часто читаемые профили, настройки, reference data.
- Результаты дорогих запросов.
- Feed pages с коротким TTL.
- Rate limit counters.

Проблемы:

- Cache stampede: использовать lock, jitter TTL, stale-while-revalidate.
- Invalidation: TTL, event-based invalidation, versioned keys.
- Hot keys: локальный in-memory cache, replication, key splitting.

Пример ключей:

```text
user:{id}:profile:v3
feed:{user_id}:cursor:{cursor_hash}
rate_limit:{user_id}:{minute}
```

## Queues и Async Processing

Очереди отделяют быстрый synchronous path от тяжелой работы.

Подходящие задачи:

- Отправка email/SMS/push.
- Генерация thumbnails.
- Индексация поиска.
- Пересчет аналитики.
- Интеграции с внешними сервисами.
- Retry payment callback handling.

PHP примеры:

- Laravel Queue jobs + Horizon для Redis/RabbitMQ/SQS.
- Symfony Messenger + transports RabbitMQ/SQS/Doctrine/Redis.
- Consumers как отдельные deployment units.

Важные свойства:

- At-least-once delivery означает, что consumer обязан быть idempotent.
- Dead-letter queue для сообщений, которые не обработались.
- Backoff retries для временных ошибок.
- Visibility timeout в SQS должен быть больше ожидаемого времени обработки.
- Poison messages не должны блокировать очередь.

## Consistency

Strong consistency нужна не везде.

Где нужна strong consistency:

- Оплата.
- Резервирование inventory.
- Балансы и ledger.
- Уникальность custom alias.

Где допустима eventual consistency:

- Аналитика кликов.
- Счетчики просмотров.
- Ленты и рекомендации.
- Уведомления.

Паттерны:

- Database transaction для локальной атомарности.
- Outbox pattern для надежной публикации событий.
- Saga для распределенных бизнес-процессов.
- Idempotency keys для повторяемых запросов.
- Optimistic locking через `version` или `updated_at`.

Пример trade-off:

> Для marketplace analytics я не буду писать каждый event синхронно в PostgreSQL. API отправит event в очередь/stream, consumer батчами пишет в ClickHouse. Это снижает latency и нагрузку, но аналитика будет обновляться с задержкой.

## Availability

Availability достигается не одной технологией, а устранением single points of failure.

Практики:

- Несколько stateless API instances.
- Load balancer health checks.
- PostgreSQL primary + replicas + backups.
- Redis Sentinel/Cluster или managed Redis.
- Очереди с HA настройками или managed SQS.
- Graceful degradation.
- Timeouts на все внешние вызовы.
- Circuit breakers для нестабильных dependencies.

Важно различать:

- High availability: система продолжает работать при отказах.
- Durability: данные не теряются.
- Fault tolerance: отказ части системы не валит все.

## Scalability

Сначала масштабируйте stateless слой, потом stateful.

Подходы:

- Horizontal scaling PHP-FPM/API containers.
- Read replicas для read-heavy PostgreSQL.
- Redis cache для hot reads.
- Queue workers autoscaling по queue depth.
- Partitioning больших таблиц.
- Sharding только когда один узел/кластер уже не справляется.
- CDN для media/static/downloads.

PHP детали:

- PHP-FPM workers ограничены CPU/RAM; считать max children.
- OPcache обязателен в production.
- Долгие задачи не выполнять в HTTP request lifecycle.
- Composer autoload optimized в production.

## Partitioning и Sharding

Partitioning - деление таблицы внутри одной логической БД. Sharding - деление данных по нескольким БД/кластерам.

Partitioning PostgreSQL:

- По времени: events, logs, analytics, audit.
- По tenant_id: multi-tenant SaaS.
- По hash user_id/order_id: равномерное распределение.

Плюсы partitioning:

- Быстрое удаление старых данных через drop partition.
- Partition pruning.
- Уменьшение индексов на partition.

Sharding:

- Использовать, когда vertical scaling, indexes, caching, replicas и partitioning уже недостаточны.
- Нужно заранее выбрать shard key.
- Cross-shard joins и transactions становятся сложными.
- Rebalancing дорогой.

Хорошие shard keys:

- `user_id` для user-centric систем.
- `tenant_id` для B2B SaaS.
- Hash от id для равномерного распределения.

Плохие shard keys:

- Timestamp, если нагрузка идет только в свежий диапазон.
- Country/region, если распределение сильно неравномерное.

## Replication

Replication повышает availability и read scalability, но добавляет lag.

Типы:

- Synchronous replication: выше consistency, ниже latency/availability.
- Asynchronous replication: выше performance, возможен replication lag.
- Logical replication: selective replication, CDC scenarios.

Применение:

- Read replicas для отчетов и тяжелых чтений.
- Hot standby для failover.
- Репликация в analytics pipeline.

Pitfall:

> После записи пользователь может не увидеть свои данные, если чтение ушло на lagging replica. Для read-your-writes можно читать с primary короткое время после write или использовать session stickiness.

## Load Balancing

Load balancer распределяет трафик и изолирует отказавшие инстансы.

Уровни:

- L4: TCP/UDP, быстро, меньше контекста.
- L7: HTTP-aware routing, headers, paths, TLS termination.

Алгоритмы:

- Round robin.
- Least connections.
- Weighted routing.
- Consistent hashing для sticky workloads.

Для PHP API обычно:

```text
Nginx/ALB/Ingress -> PHP-FPM/API containers -> services/storage
```

Важно:

- Stateless sessions, хранить session в Redis/DB/JWT при необходимости.
- Health checks должны проверять readiness, а не только process alive.
- Graceful shutdown для обработки in-flight requests.

## Rate Limiting

Rate limiting защищает систему от abuse и перегрузки.

Алгоритмы:

- Fixed window: просто, но burst на границе окна.
- Sliding window: точнее, дороже.
- Token bucket: позволяет burst в пределах bucket.
- Leaky bucket: сглаживает поток.

Где применять:

- Per IP для публичных endpoints.
- Per user/API key для authenticated API.
- Per route для дорогих операций.
- Per tenant для B2B fairness.

Redis пример:

```text
INCR rate:{api_key}:{minute}
EXPIRE rate:{api_key}:{minute} 120
```

Senior-level уточнение:

> Для distributed rate limiting важно выполнять increment+expire атомарно через Lua script или готовый middleware, иначе возможны race conditions.

## Idempotency

Idempotency означает, что повтор одного и того же запроса не приводит к повторному side effect.

Где обязательно:

- Создание платежа.
- Создание заказа.
- Отправка webhook callbacks.
- Consumers очередей.
- External provider integrations.

Реализация:

- Клиент отправляет `Idempotency-Key`.
- Сервер хранит key, request hash, response, status.
- Unique constraint на key + actor/scope.
- Повтор возвращает тот же результат.
- Если request hash отличается - вернуть conflict.

Таблица:

```sql
idempotency_keys(key, actor_id, request_hash, response_body, status_code, expires_at, created_at)
```

## Retries

Retries помогают при transient errors, но могут усилить аварию.

Правила:

- Retry только для временных ошибок: timeout, 429, 503, network reset.
- Не retry без idempotency.
- Exponential backoff + jitter.
- Ограничить max attempts.
- После лимита - DLQ/manual review.
- Не retry validation/business errors.

Пример:

```text
attempt 1: immediately
attempt 2: 5s + jitter
attempt 3: 30s + jitter
attempt 4: 5m + jitter
then DLQ
```

## Circuit Breakers

Circuit breaker защищает систему от зависимой деградирующей системы.

Состояния:

- Closed: запросы проходят.
- Open: запросы быстро отклоняются или fallback.
- Half-open: пробные запросы для восстановления.

Где полезно:

- Payment provider.
- SMS/email provider.
- External partner API.
- Search/analytics dependency.

Fallback examples:

- Вернуть cached data.
- Поставить задачу в очередь.
- Показать degraded response.
- Переключиться на второго провайдера.

## Observability

Observability отвечает на вопрос: что происходит в production и почему.

Три базовых сигнала:

- Logs: структурированные JSON logs с `trace_id`, `user_id`, `order_id`.
- Metrics: RED/USE metrics, latency percentiles, error rate, saturation.
- Traces: distributed tracing через HTTP, queue, DB calls.

Что мониторить:

- API p50/p95/p99 latency.
- 4xx/5xx rate.
- Queue depth и oldest message age.
- Worker success/failure rate.
- PostgreSQL connections, locks, slow queries, replication lag.
- Redis memory, evictions, hit ratio.
- RabbitMQ ready/unacked messages, consumer count.
- ClickHouse insert latency, failed inserts, disk usage.

Alerts должны быть actionable:

- `p95 latency > SLO 10 минут`.
- `queue oldest message age > 5 минут`.
- `payment error rate > 2%`.
- `PostgreSQL replication lag > 30 секунд`.

## Security Basics

Минимальный набор для backend system design:

- TLS everywhere.
- Authentication: session/JWT/OAuth2/OIDC по контексту.
- Authorization: RBAC/ABAC, tenant isolation.
- Input validation и output encoding.
- SQL injection защита через prepared statements/ORM query builder.
- Secrets в secret manager, не в env dumps/logs.
- Encryption at rest для sensitive data.
- Audit log для критичных действий.
- Rate limiting и WAF для публичных endpoints.
- Principle of least privilege для DB/users/service accounts.
- PII minimization и retention policies.

PHP/Laravel/Symfony:

- Не логировать токены, пароли, payment data.
- CSRF для browser forms.
- Secure cookies: HttpOnly, Secure, SameSite.
- Password hashing: Argon2id/bcrypt через framework facilities.
- Проверять serialization/deserialization risks.

## Deployment и Rollout

Хороший senior ответ включает безопасную доставку изменений.

Практики:

- Blue/green deployment.
- Canary release.
- Rolling deployment.
- Feature flags.
- Backward-compatible DB migrations.
- Expand/contract migration pattern.
- Automated rollback.
- Smoke tests после deploy.

DB migration порядок:

1. Add nullable column/new table.
2. Deploy code that writes both old and new if нужно.
3. Backfill data.
4. Switch reads to new field.
5. Stop writing old field.
6. Drop old field later.

Для PHP:

- Очистка/прогрев config cache, route cache, OPcache.
- Совместимость старых workers с новыми message schemas.
- Graceful restart consumers.

## Disaster Recovery

Disaster recovery - план восстановления после серьезного отказа.

Понятия:

- RPO: сколько данных можно потерять.
- RTO: как быстро нужно восстановиться.

Практики:

- Regular backups.
- Проверка restore, а не только наличие backup.
- PITR для PostgreSQL.
- Cross-region backup для критичных данных.
- Runbooks для failover.
- Game days/chaos testing для зрелых команд.
- DLQ replay procedures.

Пример:

> Для order/payment системы RPO должен быть близок к нулю, RTO может быть 15-60 минут в зависимости от бизнеса. Для click analytics RPO может быть несколько минут, а восстановление возможно через replay raw events.

## Trade-Offs

Интервьюеры ожидают, что вы не просто выбираете технологию, а объясняете цену выбора.

Частые trade-offs:

- Strong consistency vs availability/latency.
- Synchronous API vs async processing.
- PostgreSQL simplicity vs specialized storage.
- Cache performance vs invalidation complexity.
- Sharding scalability vs operational complexity.
- Managed services vs vendor lock-in/cost.
- Microservices autonomy vs distributed complexity.
- Real-time analytics vs batch simplicity.
- Exactly-once illusion vs practical idempotent at-least-once processing.

Фраза-шаблон:

> Я бы начал с PostgreSQL + Redis + queue, потому что это покрывает текущий масштаб и проще в эксплуатации. ClickHouse добавил бы для аналитики при росте event volume. Sharding не вводил бы до доказанной необходимости, потому что он резко усложняет транзакции, joins и rebalancing.

## Common Interview Tasks

### URL Shortener

Требования:

- Создать short URL.
- Редиректить с низкой latency.
- Custom alias, TTL, analytics.

Архитектура:

- Laravel/Symfony API для create/manage.
- PostgreSQL для links metadata.
- Redis для hot code -> URL cache.
- Queue для click events.
- ClickHouse для analytics.

Ключевые моменты:

- Unique constraint на code/custom alias.
- 301 vs 302 зависит от product requirements.
- Генерация code: base62 sequence/random с collision handling.
- Abuse protection и rate limiting.
- Click analytics async.

Pitfalls:

- Писать каждый click синхронно в PostgreSQL.
- Не учитывать hot links.
- Не иметь TTL/invalidation strategy.

### Feed

Требования:

- Показать персональную ленту.
- Поддержать follow/unfollow.
- Ранжирование, pagination, freshness.

Архитектура:

- PostgreSQL для users/posts/follows.
- Redis Sorted Sets для precomputed feed.
- Queue workers для fanout on write.
- Fallback fanout on read для heavy users.

Trade-off:

- Fanout on write: быстрые reads, дорогие writes.
- Fanout on read: простые writes, дорогие reads.
- Hybrid: celebrities/heavy users считаются при чтении.

Pitfalls:

- Offset pagination на больших таблицах.
- Не учитывать privacy changes.
- Не иметь strategy для celebrity accounts.

### Marketplace Analytics

Требования:

- События просмотров, кликов, заказов.
- Дашборды продавцов.
- Агрегации по времени, категории, кампании.

Архитектура:

- API принимает events.
- Queue/stream буферизует.
- Consumers батчами пишут в ClickHouse.
- PostgreSQL хранит users/shops/products.
- Redis кеширует популярные dashboards.

Ключевые моменты:

- Event schema versioning.
- Deduplication через event_id.
- Retention raw events и materialized views.
- Eventual consistency приемлема.

Pitfalls:

- Использовать PostgreSQL как OLAP при больших объемах.
- Не версионировать event schema.
- Не отделить raw events от aggregates.

### Notification Service

Требования:

- Email/SMS/push/webhook.
- Templates, preferences, unsubscribe.
- Retries, provider failover, delivery status.

Архитектура:

- API для enqueue notification.
- PostgreSQL для templates/preferences/status.
- RabbitMQ/SQS для per-channel queues.
- Workers на Symfony Messenger/Laravel Queue.
- Redis для rate limits и provider throttling.

Ключевые моменты:

- Idempotency key на notification request.
- DLQ и retry policy.
- Provider-specific rate limits.
- Suppression list/unsubscribe compliance.

Pitfalls:

- Синхронно отправлять SMS/email в API request.
- Retry без дедупликации.
- Игнорировать provider quotas.

### File/Media Service

Требования:

- Upload/download files.
- Metadata, access control.
- Thumbnails/transcoding.
- Virus scanning.

Архитектура:

- API выдает pre-signed upload URL.
- Object Storage хранит binary.
- PostgreSQL хранит metadata/status/owner.
- Queue запускает scan/thumbnail/transcode jobs.
- CDN для downloads/public media.

Ключевые моменты:

- Не проксировать большие файлы через PHP, если можно использовать direct upload.
- Multipart upload для больших файлов.
- Signed URLs с TTL.
- Async processing states: uploaded, scanning, ready, failed.

Pitfalls:

- Хранить файлы в PostgreSQL без необходимости.
- Не проверять content type/size.
- Давать публичные permanent URLs для private files.

### Order Processing

Требования:

- Создание заказа.
- Inventory reservation.
- Payment.
- Status transitions.
- Notifications/refunds.

Архитектура:

- Laravel/Symfony order API.
- PostgreSQL как source of truth.
- Transaction для order + reservation.
- Payment integration через idempotent calls.
- Outbox events для notifications/analytics.
- Queue workers для async side effects.

Ключевые моменты:

- State machine для order status.
- Idempotency на create order/payment.
- Saga для distributed flow.
- Compensation для failed payment/reservation expiry.
- Ledger-like модель для денег.

Pitfalls:

- Держать DB transaction во время внешнего payment API call.
- Не иметь unique idempotency key.
- Смешивать аналитические события и transactional state.

## Senior-Level Answer Templates

### Начало ответа

```text
Сначала уточню scope. Я предполагаю, что нам нужен MVP с X, Y, Z. Не включаю A и B, если интервьюер не скажет обратное. Основные NFR: p95 latency, availability, durability и expected traffic. После этого предложу baseline architecture и отдельно разберу bottlenecks.
```

### Выбор storage

```text
Для transactional source of truth я выберу PostgreSQL: нужны constraints, transactions и понятная модель данных. Redis использую как cache/rate limit/session store, но не как единственный durable storage. Для аналитики больших event volumes добавлю ClickHouse, потому что access pattern OLAP, а не OLTP.
```

### Очереди

```text
Все side effects, которые не обязаны завершиться в HTTP request, вынесу в очередь: notifications, analytics, media processing. Поскольку delivery обычно at-least-once, consumers делаю idempotent, добавляю retries with backoff и DLQ.
```

### Consistency

```text
В critical path заказа нужна strong consistency внутри PostgreSQL transaction. Межсервисные side effects публикую через outbox pattern. Для analytics и notifications принимаю eventual consistency, потому что это снижает latency и повышает resilience.
```

### Масштабирование

```text
Начну с stateless horizontal scaling API и workers, затем добавлю Redis cache и read replicas. Partitioning применю для больших time-series таблиц. Sharding оставлю как поздний шаг, потому что он сильно усложняет operational model.
```

## Pitfalls

Частые ошибки на интервью:

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

Упражнение 1: URL shortener за 20 минут.

- 3 минуты: requirements.
- 3 минуты: capacity estimates.
- 5 минут: API/data model.
- 5 минут: architecture/caching/analytics.
- 4 минуты: failure modes/trade-offs.

Упражнение 2: Notification service.

- Опишите API enqueue.
- Спроектируйте таблицы templates/preferences/status.
- Выберите RabbitMQ или SQS и объясните почему.
- Опишите retries, DLQ, idempotency, provider failover.

Упражнение 3: Order processing.

- Нарисуйте state machine заказа.
- Разделите transaction boundary и external calls.
- Добавьте outbox pattern.
- Опишите компенсации при failed payment.

Упражнение 4: Marketplace analytics.

- Оцените event volume.
- Выберите ClickHouse schema.
- Опишите ingestion pipeline.
- Объясните retention и deduplication.

## Architecture Review Checklist

Перед финальным ответом проверьте:

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

## Ссылки

- Martin Fowler - Architecture: https://martinfowler.com/architecture/
- Martin Fowler - Microservices: https://martinfowler.com/articles/microservices.html
- AWS Architecture Center: https://aws.amazon.com/architecture/
- AWS Well-Architected Framework: https://aws.amazon.com/architecture/well-architected/
- Google SRE Book: https://sre.google/sre-book/table-of-contents/
- Google SRE Workbook: https://sre.google/workbook/table-of-contents/
- Microsoft Azure Architecture Center: https://learn.microsoft.com/en-us/azure/architecture/
- Redis Documentation: https://redis.io/docs/latest/
- Redis Patterns: https://redis.io/learn/howtos/solutions
- RabbitMQ Documentation: https://www.rabbitmq.com/docs
- PostgreSQL Documentation: https://www.postgresql.org/docs/
- PostgreSQL Partitioning: https://www.postgresql.org/docs/current/ddl-partitioning.html
- ClickHouse Documentation: https://clickhouse.com/docs
- Amazon SQS Documentation: https://docs.aws.amazon.com/sqs/
- OpenAPI Specification: https://spec.openapis.org/oas/latest.html
- Designing Data-Intensive Applications, Martin Kleppmann: https://dataintensive.net/
- System Design Primer: https://github.com/donnemartin/system-design-primer
- Microsoft REST API Guidelines: https://github.com/microsoft/api-guidelines
- System Design для начинающих: всё, что вам нужно. Часть 1: https://habr.com/ru/articles/873388/