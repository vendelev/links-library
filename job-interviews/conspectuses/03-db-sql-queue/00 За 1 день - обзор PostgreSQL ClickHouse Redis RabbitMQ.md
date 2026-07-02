# За 1 день: маршрут повторения PostgreSQL, ClickHouse, Redis, RabbitMQ

Этот файл не дублирует подробные конспекты. Он нужен как маршрут на 1 день:
что повторить, в каком порядке и в какой canonical-файл идти за деталями.

## Canonical-файлы

- SQL-синтаксис, `JOIN`, группировки, оконные функции, CTE, подзапросы:
  [`01 SQL базовый и продвинутый.md`](01%20SQL%20базовый%20и%20продвинутый.md)
- PostgreSQL internals, индексы, `EXPLAIN`, MVCC, locks, WAL, VACUUM, planner:
  [`02 PostgreSQL production internals.md`](02%20PostgreSQL%20production%20internals.md)
- Масштабирование БД, partitioning, replication, sharding, consistency, DR:
  [`03 Масштабирование баз данных.md`](03%20Масштабирование%20баз%20данных.md)
- ClickHouse, MergeTree, materialized views, ingestion, distributed tables:
  [`04 ClickHouse.md`](04%20ClickHouse.md)
- Redis, caching, TTL, eviction, locks, Streams, persistence, Sentinel, Cluster:
  [`05 Redis.md`](05%20Redis.md)
- RabbitMQ/SQS, ack/nack, prefetch, retry, DLQ, idempotency, backpressure:
  [`06 RabbitMQ и SQS.md`](06%20RabbitMQ%20и%20SQS.md)

## Как готовиться за 1 день

1. Пройти SQL-ядро: логический порядок `SELECT`, `JOIN`, `GROUP BY`, оконные функции, CTE.
2. Повторить PostgreSQL production-темы: индексы, `EXPLAIN`, транзакции, MVCC, locks, VACUUM, WAL.
3. Повторить scaling: read/write scaling, partitioning, replication, sharding, lag, failover.
4. Пройти ClickHouse: когда использовать, `MergeTree`, `PARTITION BY`, `ORDER BY`, MV, ingestion.
5. Пройти Redis: cache-aside, TTL/jitter, stampede, locks, Streams, persistence, Cluster.
6. Пройти RabbitMQ/SQS: exchange/queue, ack, prefetch, retries, DLQ, idempotency, backpressure.
7. В конце проговорить вслух short answers и решить mini-practice из каждого canonical-файла.

## Что обязательно уметь объяснить

### SQL

- Почему `WHERE` не видит агрегаты и оконные функции того же уровня.
- Чем `WHERE` отличается от `HAVING`.
- Почему `LEFT JOIN` может стать фактическим `INNER JOIN`.
- Когда использовать `EXISTS`, а когда `JOIN`.
- Почему `NOT IN` с `NULL` опасен.
- Чем отличаются `ROW_NUMBER`, `RANK`, `DENSE_RANK`.
- Почему CTE не обязательно быстрее подзапроса.

### PostgreSQL

- Почему индекс может не использоваться.
- Как читать `EXPLAIN (ANALYZE, BUFFERS)`.
- Что означают estimated rows vs actual rows.
- Как MVCC связан с долгими транзакциями, bloat и VACUUM.
- Чем отличаются `READ COMMITTED`, `REPEATABLE READ`, `SERIALIZABLE`.
- Как работают row locks, advisory locks и `FOR UPDATE SKIP LOCKED`.
- Как предотвращать и обрабатывать deadlocks/serialization failures.
- Зачем нужен WAL и почему commit не обязан сразу писать data pages на диск.

### Масштабирование

- Когда достаточно индексов, query plans, pooling и vertical scaling.
- Чем partitioning отличается от sharding.
- Почему read replicas не масштабируют запись.
- Чем опасен replication lag и как решать read-after-write.
- Как выбрать shard key.
- Почему cross-shard transactions лучше избегать.
- Что такое RPO/RTO, failover, split-brain, backup restore test.

### ClickHouse

- Когда ClickHouse лучше PostgreSQL для аналитики.
- Почему ClickHouse не стоит использовать как основной OLTP-store.
- Чем `PARTITION BY` отличается от `ORDER BY`.
- Почему `PRIMARY KEY` не означает уникальность.
- Почему materialized view не решает backfill автоматически.
- Почему insert нужно делать батчами.
- Что делают `ReplicatedMergeTree` и `Distributed`.

### Redis

- Когда Redis хорош как кеш, а когда опасен как source of truth.
- Что такое cache-aside, TTL, jitter, stampede, eviction.
- Как безопасно брать и освобождать lock через `SET NX PX` и Lua unlock.
- Почему Redlock не silver bullet.
- Чем Pub/Sub отличается от Streams.
- Что дают RDB, AOF, replication, Sentinel, Cluster.
- Почему multi-key операции в Redis Cluster требуют hash tags.

### RabbitMQ/SQS

- Чем exchange отличается от queue.
- Почему ack должен быть после durable side effect.
- Чем опасен `nack(requeue=true)` для poison message.
- Как проектировать retry с backoff, jitter и DLQ.
- Почему consumer должен быть идемпотентным.
- Что означает prefetch и как он связан с backpressure.
- Чем SQS visibility timeout отличается от RabbitMQ ack model.
- Когда нужна очередность по aggregate id, а не глобальный ordering.

## Финальный прогон перед интервью

1. Открыть `01` и решить mini-practice по SQL-запросам.
2. Открыть `02` и проговорить PostgreSQL senior answers.
3. Открыть `03` и проговорить system design шаблон масштабирования.
4. Открыть `04`, `05`, `06` и пройти checklist перед production.
5. На каждый вопрос отвечать через trade-off: когда подходит, когда нет, какие failure modes, что мониторить.

## Ссылки на документацию

Подробные ссылки на документацию и статьи находятся в canonical-файлах:

- PostgreSQL, MySQL, ClickHouse SQL-документация: `01 SQL базовый и продвинутый.md`.
- PostgreSQL MVCC, VACUUM, WAL, planner, indexes: `02 PostgreSQL production internals.md`.
- PostgreSQL partitioning, replication, sharding, DR и cloud docs: `03 Масштабирование баз данных.md`.
- ClickHouse MergeTree, Distributed, replication, materialized views: `04 ClickHouse.md`.
- Redis commands, data types, locks, streams, persistence, Cluster: `05 Redis.md`.
- RabbitMQ, SQS, Laravel Queues, Symfony Messenger: `06 RabbitMQ и SQS.md`.
