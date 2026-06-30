# За 1 день: PostgreSQL/ClickHouse/Redis/RabbitMQ, индексы, транзакции, блокировки

Гайд для Lead/Senior PHP developer, которому нужно быстро освежить практические темы для собеседования: как объяснять решения, где бывают ловушки и какие ответы ожидаются на senior-уровне.

## Как готовиться за 1 день

1. PostgreSQL: индексы, `EXPLAIN`, транзакции, блокировки, MVCC.
2. ClickHouse: MergeTree, `PARTITION BY`, `ORDER BY`, материализованные представления, ingestion.
3. Redis: структуры данных, кеширование, TTL/eviction, locks, streams/pubsub, persistence.
4. RabbitMQ: exchange/queue/binding, ack/nack, prefetch, retry/DLQ, идемпотентность.
5. Проговорить вслух short answers и разобрать мини-практику.

## PostgreSQL

### Индексы

Индекс ускоряет чтение ценой дополнительной записи, места на диске и обслуживания статистики. Senior-ответ: индекс выбирают под конкретный запрос, кардинальность, селективность, сортировку, фильтры и профиль записи.

#### B-tree

Основной индекс по умолчанию.

Подходит для:

- `=`, `<`, `>`, `BETWEEN`, `IN`;
- `ORDER BY`;
- prefix-поиска `LIKE 'abc%'` при подходящей collation/operator class;
- уникальности через `UNIQUE`.

Пример:

```sql
CREATE INDEX idx_orders_user_created_at ON orders (user_id, created_at DESC);
```

Практическое объяснение: индекс `(user_id, created_at)` хорошо работает для `WHERE user_id = ? ORDER BY created_at DESC`, но не обязан помогать запросу только по `created_at`, потому что нарушается leftmost-prefix rule.

#### GIN

Generalized Inverted Index. Используется для составных значений, где один документ содержит много ключей/токенов.

Подходит для:

- `jsonb`;
- массивов;
- full-text search;
- `@>`, `?`, `?|`, `?&`.

```sql
CREATE INDEX idx_events_payload_gin ON events USING gin (payload jsonb_path_ops);
```

Ловушка: GIN обычно тяжелее на запись и занимает больше места. Для `jsonb_path_ops` быстрее containment `@>`, но меньше поддерживаемых операторов, чем у стандартного `jsonb_ops`.

#### GiST

Generalized Search Tree. Гибкий индекс для геометрии, range types, nearest-neighbor search, exclusion constraints.

```sql
CREATE INDEX idx_bookings_period_gist ON bookings USING gist (period);
```

Типовой кейс: запрет пересекающихся броней через exclusion constraint.

```sql
ALTER TABLE bookings
ADD CONSTRAINT no_room_overlap
EXCLUDE USING gist (room_id WITH =, period WITH &&);
```

#### BRIN

Block Range Index. Очень компактный индекс, хранит summary по диапазонам страниц.

Подходит для больших append-only таблиц, где данные физически коррелируют с колонкой: timestamp, sequence id.

```sql
CREATE INDEX idx_logs_created_brin ON logs USING brin (created_at);
```

Ловушка: BRIN не заменяет B-tree для точечного поиска. Он хорош, когда можно отбросить большие диапазоны блоков.

#### Partial Index

Индекс только по части строк.

```sql
CREATE INDEX idx_orders_unpaid ON orders (created_at)
WHERE status = 'unpaid';
```

Подходит для частых запросов по маленькому активному подмножеству: `deleted_at IS NULL`, `status = 'pending'`, `processed_at IS NULL`.

Ловушка: планировщик использует partial index только если условие запроса логически совпадает с predicate индекса.

#### Composite Index

Многоколоночный индекс.

```sql
CREATE INDEX idx_payments_tenant_status_created
ON payments (tenant_id, status, created_at DESC);
```

Правило: сначала равенства и tenant/filter columns, затем range/sort columns. Но всегда проверять через `EXPLAIN (ANALYZE, BUFFERS)`.

#### Covering Index

Индекс с `INCLUDE`, который позволяет index-only scan, если visibility map позволяет не читать heap.

```sql
CREATE INDEX idx_users_email_include
ON users (email)
INCLUDE (id, name, created_at);
```

Ловушка: `INCLUDE`-колонки не участвуют в сортировке и поиске, они только покрывают выборку.

### EXPLAIN

Используй:

```sql
EXPLAIN (ANALYZE, BUFFERS, VERBOSE)
SELECT * FROM orders WHERE user_id = 42 ORDER BY created_at DESC LIMIT 20;
```

Что смотреть:

- `actual time`, `rows`, `loops`;
- отличие estimated rows от actual rows;
- `Seq Scan`, `Index Scan`, `Index Only Scan`, `Bitmap Heap Scan`;
- `Sort`, `Hash Join`, `Nested Loop`, `Merge Join`;
- `Buffers: shared hit/read/dirtied`;
- `Rows Removed by Filter`;
- не только cost, а фактическое время и I/O.

Senior-ответ: `Seq Scan` не всегда плохо. Для маленькой таблицы или большой доли строк он может быть быстрее индекса.

Частые причины плохого плана:

- устаревшая статистика, нужен `ANALYZE`;
- низкая селективность условия;
- функция на колонке без expression index: `LOWER(email)`;
- неверный порядок колонок в composite index;
- parameter sniffing/ generic plan в prepared statements;
- collation/operator class не подходит для `LIKE`.

### Транзакции

ACID:

- Atomicity: все или ничего;
- Consistency: ограничения БД не нарушены;
- Isolation: параллельные транзакции не ломают друг друга;
- Durability: committed данные переживают сбой согласно настройкам WAL/fsync.

```sql
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
```

Ловушки:

- долгие транзакции удерживают старые версии строк и мешают vacuum;
- нельзя держать транзакцию во время внешних HTTP-вызовов;
- в PHP важно явно контролировать `beginTransaction`, `commit`, `rollBack` и не оставлять соединение в failed transaction state.

### Уровни изоляции

PostgreSQL поддерживает:

- `READ COMMITTED`: default, каждый statement видит новый committed snapshot;
- `REPEATABLE READ`: вся транзакция видит один snapshot, защищает от non-repeatable read и phantom read в PostgreSQL snapshot isolation;
- `SERIALIZABLE`: strongest, Serializable Snapshot Isolation, возможны serialization failures, нужен retry;
- `READ UNCOMMITTED`: фактически работает как `READ COMMITTED`.

Практический senior-ответ: повышать изоляцию нужно осознанно. Часто лучше использовать row-level locks, unique constraints, idempotency keys и retry на конфликт.

Пример retry для `SERIALIZABLE`/deadlock:

```php
for ($attempt = 1; $attempt <= 3; $attempt++) {
    try {
        $pdo->beginTransaction();
        // business changes
        $pdo->commit();
        break;
    } catch (Throwable $e) {
        if ($pdo->inTransaction()) {
            $pdo->rollBack();
        }
        if ($attempt === 3 || !isRetryableDbError($e)) {
            throw $e;
        }
        usleep(random_int(10_000, 50_000));
    }
}
```

### MVCC

MVCC хранит несколько версий строк. Читатели не блокируют писателей, писатели не блокируют обычных читателей. Каждая транзакция видит snapshot.

Что важно знать:

- `UPDATE` создает новую версию строки;
- старые версии чистит `VACUUM`;
- долгие транзакции мешают очистке;
- bloat растет при частых update/delete;
- autovacuum нужно мониторить, а не отключать.

### Блокировки

Типовые уровни:

- table-level locks: DDL, `LOCK TABLE`, некоторые операции maintenance;
- row-level locks: `SELECT FOR UPDATE`, `UPDATE`, `DELETE`;
- advisory locks: прикладные именованные locks;
- predicate locks: для `SERIALIZABLE`.

Полезные конструкции:

```sql
SELECT * FROM jobs
WHERE status = 'pending'
ORDER BY id
FOR UPDATE SKIP LOCKED
LIMIT 10;
```

Использование: конкурентные воркеры берут задачи без ожидания уже заблокированных строк.

```sql
SELECT * FROM invoices WHERE id = 10 FOR UPDATE NOWAIT;
```

Использование: быстро упасть, если строка уже заблокирована.

Ловушки:

- `SELECT FOR UPDATE` блокирует выбранные строки до конца транзакции;
- разные порядки обновления ресурсов повышают риск deadlock;
- DDL может ждать долгие транзакции и блокировать прод;
- advisory locks не заменяют constraints.

### Deadlocks

Deadlock: транзакция A держит ресурс 1 и ждет ресурс 2, транзакция B держит ресурс 2 и ждет ресурс 1. PostgreSQL обнаружит deadlock и прервет одну транзакцию.

Профилактика:

- обновлять ресурсы в одинаковом порядке;
- держать транзакции короткими;
- не делать внешние вызовы внутри транзакции;
- использовать retry для deadlock/serialization failures;
- брать блокировки как можно позднее, но до проверки критичного состояния.

### Constraints

Constraints - это защита инвариантов на уровне БД.

- `PRIMARY KEY`;
- `FOREIGN KEY`;
- `UNIQUE`;
- `CHECK`;
- `NOT NULL`;
- `EXCLUDE`;
- `DEFERRABLE` constraints.

Senior-ответ: валидация в PHP нужна для UX, constraints в БД нужны для целостности при конкуренции, импортах и баге в приложении.

Пример идемпотентности:

```sql
CREATE UNIQUE INDEX uniq_payments_idempotency_key
ON payments (idempotency_key)
WHERE idempotency_key IS NOT NULL;
```

## ClickHouse

### Когда использовать

ClickHouse - колонковая OLAP-СУБД для быстрых аналитических запросов по большим объемам append-heavy данных.

Хорошо подходит для:

- событий, логов, метрик;
- аналитики по временным рядам;
- агрегаций по миллионам/миллиардам строк;
- дешевого сканирования нужных колонок.

Плохо подходит для:

- OLTP с частыми point update/delete;
- строгих транзакций как в PostgreSQL;
- частых single-row lookups;
- уникальных constraints и FK как основного механизма консистентности;
- сценариев, где данные постоянно мутируют.

### MergeTree Basics

Семейство `MergeTree` - основной движок таблиц. Данные пишутся частями, затем background merges объединяют parts.

```sql
CREATE TABLE events
(
    event_date Date,
    event_time DateTime,
    user_id UInt64,
    event_name LowCardinality(String),
    properties String
)
ENGINE = MergeTree
PARTITION BY toYYYYMM(event_date)
ORDER BY (event_name, event_date, user_id);
```

Важные понятия:

- part: кусок данных на диске;
- partition: логическое разбиение для управления данными;
- primary key: в ClickHouse связан с sparse index и обычно совпадает или является prefix от `ORDER BY`;
- `ORDER BY`: физический порядок данных, критичен для skipping;
- granule: минимальный блок индексирования.

### PARTITION BY, ORDER BY, PRIMARY KEY

`PARTITION BY` нужен не для ускорения каждого запроса, а для lifecycle операций: drop partition, TTL, управление объемом parts.

Хороший partition key:

- не слишком мелкий;
- обычно месяц/день для time-series;
- соответствует retention и загрузке.

Ловушка: `PARTITION BY user_id` при миллионах пользователей создаст слишком много partitions и parts.

`ORDER BY` выбирается под самые частые фильтры и группировки. Колонки с высокой селективностью и частым фильтром обычно ставят раньше, но нужно учитывать реальные запросы.

Senior-ответ: в ClickHouse `PRIMARY KEY` не гарантирует уникальность. Это sparse index для ускорения чтения.

### Материализованные представления

Materialized View в ClickHouse обычно используется как insert-trigger: при вставке в source table данные трансформируются и пишутся в target table.

```sql
CREATE TABLE events_daily
(
    event_date Date,
    event_name LowCardinality(String),
    cnt UInt64
)
ENGINE = SummingMergeTree
PARTITION BY toYYYYMM(event_date)
ORDER BY (event_date, event_name);

CREATE MATERIALIZED VIEW mv_events_daily
TO events_daily AS
SELECT event_date, event_name, count() AS cnt
FROM events
GROUP BY event_date, event_name;
```

Ловушки:

- MV не пересчитывает старые данные автоматически;
- backfill нужно делать отдельно;
- агрегационные движки требуют понимать merge semantics;
- `FINAL` может быть дорогим.

### Ingestion

Рекомендации:

- вставлять батчами, а не по одной строке;
- избегать слишком большого количества small inserts;
- использовать Kafka engine/Buffer/async inserts при необходимости;
- следить за количеством parts;
- проектировать schema под запросы, а не нормализовать как OLTP.

PHP-практика: для событий лучше буферизовать и писать пачками через worker, чем синхронно вставлять каждое событие из web request.

### Когда не использовать ClickHouse

Не стоит выбирать ClickHouse, если нужны:

- банковские транзакции с сильной консистентностью;
- частые `UPDATE`/`DELETE` по одной строке;
- сложные FK и constraints;
- основной источник истины для mutable бизнес-объектов;
- низкая задержка для большого числа point lookups по PK.

## Redis

### Структуры данных

Redis - in-memory data store с разными структурами данных.

- String: кеш значения, счетчики, distributed lock token;
- Hash: объект с полями, user/session profile;
- List: простая очередь, но для надежных очередей лучше Streams/RabbitMQ;
- Set: уникальные элементы, membership;
- Sorted Set: рейтинг, delayed jobs, time-indexed data;
- Stream: append-only log для consumer groups;
- Bitmap/HyperLogLog: компактная статистика.

Senior-ответ: Redis - не просто key-value. Выбор структуры влияет на сложность операций, память и атомарность.

### Caching Patterns

Cache-aside:

1. Приложение читает Redis.
2. При miss читает БД.
3. Пишет результат в Redis с TTL.

```php
$key = "user:$id";
$cached = $redis->get($key);
if ($cached !== false) {
    return json_decode($cached, true);
}

$user = $repository->find($id);
$redis->setex($key, 300, json_encode($user));
return $user;
```

Write-through: запись идет через кеш и storage. Проще читать, сложнее гарантировать консистентность.

Write-behind: запись сначала в кеш/буфер, потом асинхронно в storage. Быстро, но риск потери данных.

Read-through: кеш сам загружает данные, чаще реализуется библиотекой/слоем.

### TTL, Stampede, Eviction

TTL нужен почти всегда, чтобы кеш сам восстанавливался после ошибок инвалидации.

Ловушки:

- cache stampede: много запросов одновременно пересчитывают один ключ;
- dogpile effect: истекают тысячи ключей одновременно;
- stale data: пользователь видит старые данные;
- unbounded keys: память растет без контроля.

Защита:

- random jitter к TTL;
- mutex на пересчет;
- stale-while-revalidate;
- negative caching для частых miss;
- versioned keys: `user:v2:123`.

Eviction policies:

- `noeviction`: ошибки записи при нехватке памяти;
- `allkeys-lru`, `volatile-lru`;
- `allkeys-lfu`, `volatile-lfu`;
- `allkeys-random`, `volatile-random`;
- `volatile-ttl`.

Senior-ответ: eviction - не замена TTL. Политика eviction должна соответствовать роли Redis: кеш, session store, rate limiter.

### Redis Locks

Минимальный lock:

```redis
SET lock:invoice:123 token NX PX 30000
```

Освобождать lock нужно только владельцем через Lua compare-and-delete.

```lua
if redis.call('get', KEYS[1]) == ARGV[1] then
    return redis.call('del', KEYS[1])
end
return 0
```

Ловушки:

- lock без TTL может зависнуть навсегда;
- нельзя просто `DEL`, можно удалить чужой lock;
- lock timeout должен быть больше ожидаемой работы;
- для критичной консистентности лучше БД constraints/transactions;
- Redlock спорен для систем с жесткими требованиями консистентности.

### Streams и Pub/Sub

Pub/Sub:

- fire-and-forget;
- сообщения не сохраняются для offline consumer;
- хорошо для live notifications.

Streams:

- persistent append-only stream;
- consumer groups;
- pending entries;
- ack через `XACK`;
- можно перечитывать и восстанавливать обработку.

Пример:

```redis
XADD orders * order_id 123 status paid
XGROUP CREATE orders order-workers $ MKSTREAM
XREADGROUP GROUP order-workers worker-1 COUNT 10 BLOCK 5000 STREAMS orders >
XACK orders order-workers 1700000000000-0
```

Ловушка: Streams не заменяют RabbitMQ во всех сценариях. RabbitMQ сильнее в routing, backpressure, retry/DLQ и интеграции messaging patterns.

### Persistence

RDB:

- snapshot через интервалы;
- быстрый restart;
- можно потерять последние изменения.

AOF:

- append-only log;
- меньше потерь при `appendfsync everysec`;
- требует rewrite.

Важно: Redis с persistence все равно не становится PostgreSQL. Нужно понимать допустимую потерю данных, replication lag и failover behavior.

## RabbitMQ

### Exchanges, Queues, Bindings

Producer публикует сообщение в exchange. Exchange маршрутизирует в queues по bindings. Consumer читает из queue.

Типы exchange:

- direct: routing key exact match;
- topic: pattern routing, например `order.*`, `order.created`;
- fanout: broadcast во все bound queues;
- headers: routing по headers.

Базовый пример:

```text
producer -> exchange orders.topic -> binding order.created -> queue billing.order_created
```

Senior-ответ: producer не должен знать конкретные queues. Он публикует событие в exchange с routing key.

### Ack/Nack

Manual ack означает: сообщение удаляется из очереди только после успешной обработки.

- `ack`: обработано успешно;
- `nack/reject requeue=true`: вернуть в очередь;
- `nack/reject requeue=false`: выбросить или отправить в DLX, если настроен;
- auto-ack опасен для важных задач.

Ловушка: если consumer упал до ack, RabbitMQ переотдаст сообщение. Поэтому обработчик обязан быть идемпотентным.

### Prefetch и Backpressure

`prefetch` ограничивает количество unacked сообщений на consumer.

```text
prefetch = 10
```

Практический смысл: не забирать из очереди больше сообщений, чем consumer реально может обработать. Для тяжелых задач prefetch часто маленький, для быстрых I/O-задач может быть больше.

Ловушка: слишком большой prefetch ломает fair dispatch и увеличивает latency для сообщений, которые уже забраны медленным consumer.

### Retries и DLQ

Простая стратегия:

- transient error: retry с задержкой;
- permanent error: DLQ;
- poison message: не гонять бесконечно.

Паттерны retry:

- retry queue с TTL и dead-letter обратно в main exchange;
- delayed message exchange plugin;
- retry count в headers;
- exponential backoff через несколько retry queues.

DLQ нужна для анализа и ручной/автоматической переобработки.

Ловушки:

- бесконечный `nack requeue=true` создает hot loop;
- retry без лимита маскирует баги;
- DLQ без мониторинга становится кладбищем сообщений;
- порядок сообщений ломается при retry.

### Идемпотентность

RabbitMQ гарантирует at-least-once delivery в типичной надежной конфигурации. Значит, дубликаты возможны.

Подходы:

- idempotency key в сообщении;
- таблица processed messages с unique constraint;
- бизнес-операции через upsert;
- проверка статуса перед изменением;
- outbox/inbox patterns.

Пример:

```sql
CREATE TABLE processed_messages (
    message_id text PRIMARY KEY,
    processed_at timestamptz NOT NULL DEFAULT now()
);
```

### Ordering

Порядок гарантируется только внутри одной queue при одном consumer и без retry/requeue. В реальных системах порядок легко нарушается из-за concurrency, redelivery, multiple consumers, retry queues.

Если нужен порядок по aggregate:

- routing key по `aggregate_id`;
- consistent hash exchange;
- partitioned queues;
- один consumer на partition;
- sequence/version check в обработчике.

Senior-ответ: глобальный порядок дорогой и редко нужен. Обычно нужен порядок внутри одного aggregate.

## Частые Собеседовательные Ловушки

- Индекс не обязан использоваться, если селективность низкая или статистика считает seq scan дешевле.
- `PRIMARY KEY` в ClickHouse не означает уникальность.
- Redis lock без token и Lua unlock небезопасен.
- RabbitMQ consumer должен быть идемпотентным, потому что delivery обычно at-least-once.
- `SERIALIZABLE` в PostgreSQL требует retry logic.
- Долгая транзакция в PostgreSQL может создать bloat и мешать vacuum.
- `SELECT FOR UPDATE SKIP LOCKED` подходит для job queue, но нужно следить за starvation.
- ClickHouse не стоит использовать как основной OLTP-store.
- Redis Pub/Sub не хранит сообщения для отключенных consumer.
- DLQ без алертов и процесса разбора не решает проблему.

## Short Senior-Level Answers

**Почему PostgreSQL не использует мой индекс?**

Потому что планировщик считает другой план дешевле: низкая селективность, маленькая таблица, устаревшая статистика, неподходящий порядок колонок, функция на колонке, mismatch типов/collation или запрос возвращает слишком большую долю таблицы. Проверяю через `EXPLAIN (ANALYZE, BUFFERS)`.

**Чем `READ COMMITTED` отличается от `REPEATABLE READ`?**

В `READ COMMITTED` каждый statement видит новый snapshot committed данных. В `REPEATABLE READ` вся транзакция видит один snapshot, поэтому повторное чтение стабильно, но при конфликтах возможны ошибки, которые нужно обрабатывать.

**Как бороться с deadlock?**

Короткие транзакции, единый порядок блокировок, отсутствие внешних вызовов внутри транзакции, точечные locks и retry на deadlock error.

**Как выбрать `ORDER BY` в ClickHouse?**

По основным фильтрам и группировкам запросов. `ORDER BY` задает физическую сортировку и sparse index, поэтому от него зависит data skipping. Нельзя выбирать только по красоте схемы.

**Почему Redis не всегда подходит для lock?**

Redis lock зависит от TTL, времени выполнения, failover и корректного unlock. Для строгих денежных инвариантов лучше использовать транзакции и constraints в БД.

**Как сделать RabbitMQ обработчик надежным?**

Manual ack после успешной обработки, idempotency key, retry с лимитом, DLQ, prefetch под нагрузку, мониторинг unacked/ready/DLQ и сохранение side effects транзакционно, например через outbox/inbox.

## Мини-Практика и Чеклисты

### PostgreSQL Index Review

Дано:

```sql
SELECT id, total
FROM orders
WHERE tenant_id = 7 AND status = 'paid'
ORDER BY created_at DESC
LIMIT 50;
```

Возможный индекс:

```sql
CREATE INDEX idx_orders_tenant_status_created
ON orders (tenant_id, status, created_at DESC)
INCLUDE (total);
```

Проверить:

- селективность `tenant_id/status`;
- нужен ли partial index по `status = 'paid'`;
- совпадает ли сортировка;
- есть ли index-only scan;
- не слишком ли дорог индекс для write-heavy таблицы.

### PostgreSQL Transaction Checklist

- Транзакция короткая.
- Нет HTTP/RPC внутри транзакции.
- Есть retry для deadlock/serialization failure.
- Инварианты закреплены constraints.
- Lock берется в понятном порядке.
- Ошибка внутри транзакции приводит к rollback.

### ClickHouse Table Design Checklist

- Понятны топ-5 запросов.
- `ORDER BY` выбран под фильтрацию и агрегации.
- `PARTITION BY` не создает тысячи мелких partitions.
- Insert идет батчами.
- Есть стратегия retention/TTL.
- Backfill для MV продуман отдельно.

### Redis Cache Checklist

- У каждого cache key есть TTL.
- Есть jitter для массовых ключей.
- Есть защита от stampede.
- Понятна eviction policy.
- Key naming versioned и предсказуемый.
- Критичные данные не живут только в Redis без осознанного риска.

### RabbitMQ Consumer Checklist

- Manual ack включен.
- Ack только после успешного side effect.
- Handler идемпотентен.
- Prefetch настроен.
- Retry имеет лимит и backoff.
- DLQ мониторится.
- Poison messages не крутятся бесконечно.

## Вопросы Для Самопроверки

1. Когда B-tree индекс `(a, b, c)` поможет запросу по `b` без `a`?
2. Чем `Index Scan` отличается от `Bitmap Heap Scan`?
3. Почему `EXPLAIN` без `ANALYZE` может вводить в заблуждение?
4. Что происходит с row version при `UPDATE` в PostgreSQL?
5. Почему long transaction опасна для autovacuum?
6. Как избежать deadlock при переводе денег между счетами?
7. Почему ClickHouse `PRIMARY KEY` не гарантирует уникальность?
8. Чем `PARTITION BY` отличается от `ORDER BY` в MergeTree?
9. Почему materialized view в ClickHouse не решает backfill автоматически?
10. Как защититься от cache stampede в Redis?
11. Почему Redis Pub/Sub нельзя использовать для надежной очереди задач?
12. Как безопасно освободить Redis lock?
13. Что означает RabbitMQ prefetch?
14. Почему `nack requeue=true` может создать проблему?
15. Как обеспечить идемпотентность consumer?
16. Когда порядок сообщений в RabbitMQ не гарантируется?

## Короткие Ответы Для Самопроверки

1. Обычно не поможет эффективно из-за leftmost-prefix rule, если нет skip scan/других условий и планировщик не видит выгоды.
2. `Index Scan` ходит по индексу и heap построчно, `Bitmap Heap Scan` сначала строит bitmap подходящих страниц, затем читает heap пачками.
3. Он показывает оценку, а не фактические строки, время и buffers.
4. Создается новая версия строки, старая остается до очистки vacuum.
5. Она удерживает старый snapshot и мешает удалять dead tuples.
6. Всегда блокировать счета в одном порядке, например по возрастанию id, и иметь retry.
7. Это sparse index для чтения, а не constraint уникальности.
8. `PARTITION BY` управляет разбиением parts/lifecycle, `ORDER BY` задает физическую сортировку и data skipping.
9. MV срабатывает на новые вставки; старые данные нужно переливать отдельно.
10. Mutex, jitter TTL, stale-while-revalidate, request coalescing.
11. Offline consumer потеряет сообщения; нет ack/retry/DLQ как в broker.
12. Через token и Lua compare-and-delete.
13. Максимум unacked сообщений, которые broker отдаст consumer.
14. Poison message может бесконечно переобрабатываться и грузить систему.
15. Idempotency key, unique constraint, processed_messages, upsert/status checks.
16. При нескольких consumers, retry/requeue, нескольких queues/partitions и асинхронной обработке.

## Ссылки

- PostgreSQL Documentation: https://www.postgresql.org/docs/current/
- PostgreSQL Indexes: https://www.postgresql.org/docs/current/indexes.html
- PostgreSQL EXPLAIN: https://www.postgresql.org/docs/current/using-explain.html
- PostgreSQL Transaction Isolation: https://www.postgresql.org/docs/current/transaction-iso.html
- PostgreSQL Explicit Locking: https://www.postgresql.org/docs/current/explicit-locking.html
- PostgreSQL MVCC: https://www.postgresql.org/docs/current/mvcc.html
- ClickHouse Documentation: https://clickhouse.com/docs
- ClickHouse MergeTree: https://clickhouse.com/docs/engines/table-engines/mergetree-family/mergetree
- ClickHouse Materialized Views: https://clickhouse.com/docs/materialized-view
- ClickHouse Best Practices: https://clickhouse.com/docs/best-practices
- Redis Documentation: https://redis.io/docs/latest/
- Redis Data Types: https://redis.io/docs/latest/develop/data-types/
- Redis Persistence: https://redis.io/docs/latest/operate/oss_and_stack/management/persistence/
- Redis Distributed Locks: https://redis.io/docs/latest/develop/clients/patterns/distributed-locks/
- Redis Streams: https://redis.io/docs/latest/develop/data-types/streams/
- RabbitMQ Documentation: https://www.rabbitmq.com/docs
- RabbitMQ Tutorials: https://www.rabbitmq.com/tutorials
- RabbitMQ Consumer Acknowledgements: https://www.rabbitmq.com/docs/confirms
- RabbitMQ Consumer Prefetch: https://www.rabbitmq.com/docs/consumer-prefetch
- RabbitMQ Dead Letter Exchanges: https://www.rabbitmq.com/docs/dlx
- Use The Index, Luke: https://use-the-index-luke.com/
- Markus Winand, SQL Performance Explained: https://sql-performance-explained.com/
- Jepsen Redis analyses: https://jepsen.io/analyses/redis-raft-1b3fbf6
- Martin Kleppmann, How to do distributed locking: https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html
