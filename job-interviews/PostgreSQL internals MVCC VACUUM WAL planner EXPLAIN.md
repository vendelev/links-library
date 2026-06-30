# PostgreSQL internals: MVCC, VACUUM, WAL, planner, EXPLAIN

Гайд для подготовки к собеседованию Lead/Senior PHP backend developer. Фокус: как PostgreSQL ведет себя под production-нагрузкой, почему запросы деградируют, как объяснять internals без ухода в академизм и как диагностировать проблемы.

## Что должен уметь Senior/Lead

- Объяснить MVCC, tuple versions, snapshots и visibility простыми словами.
- Понимать, почему `UPDATE` создает новую версию строки, а не меняет ее in-place.
- Отличать row locks, table locks, predicate locks и advisory locks.
- Читать `EXPLAIN (ANALYZE, BUFFERS)` и видеть ошибку в оценках planner-а.
- Диагностировать bloat, долгие транзакции, autovacuum lag, lock contention, deadlocks.
- Понимать роль WAL, checkpoints, replication lag и durability-настроек.
- Знать, как статистика и индексы влияют на план запроса.
- Уметь связать проблемы PostgreSQL с PHP-приложением: транзакции, connection pool, ORM, N+1, batch jobs.

## MVCC: основная идея

MVCC, Multi-Version Concurrency Control, позволяет читателям не блокировать писателей, а писателям не блокировать читателей в обычном случае.

PostgreSQL хранит несколько версий одной логической строки. Каждая версия называется tuple. Когда транзакция обновляет строку, PostgreSQL не перезаписывает старую версию, а создает новую tuple-version. Старые версии остаются в таблице, пока они могут быть видимы каким-то snapshot-ам, а потом удаляются VACUUM-ом.

Ключевая мысль для интервью:

> PostgreSQL обеспечивает конкурентность не через чтение старых данных из undo-log, а через хранение версий строк прямо в heap-таблице. Snapshot решает, какие версии видимы конкретной транзакции.

## Tuple versions и системные поля

У каждой tuple есть служебная информация:

- `xmin` - ID транзакции, которая создала эту версию строки.
- `xmax` - ID транзакции, которая удалила или заменила эту версию строки; если версия актуальна, обычно пустой/нулевой.
- `ctid` - физический адрес tuple: страница и позиция внутри страницы.
- hint bits - подсказки о commit/abort статусе транзакций, чтобы не ходить каждый раз в commit log.

Пример:

```sql
CREATE TABLE accounts (
    id bigint PRIMARY KEY,
    balance numeric NOT NULL
);

INSERT INTO accounts VALUES (1, 100);

SELECT xmin, xmax, ctid, * FROM accounts WHERE id = 1;

UPDATE accounts SET balance = 150 WHERE id = 1;
SELECT xmin, xmax, ctid, * FROM accounts WHERE id = 1;
```

После `UPDATE` появится новая физическая версия строки. Старая версия станет dead tuple после того, как перестанет быть видимой активным snapshot-ам.

## HOT updates

HOT, Heap-Only Tuple, это оптимизация `UPDATE`, когда PostgreSQL может не обновлять индексные записи.

HOT возможен, если:

- изменились только поля, которые не входят в индексы;
- на heap page есть свободное место для новой версии tuple;
- fillfactor оставляет место под обновления.

Почему это важно:

- меньше записей в WAL;
- меньше index bloat;
- быстрее `UPDATE`;
- меньше работы для VACUUM.

Пример production-вывода:

> Если таблица часто обновляется, я смотрю, какие поля обновляются, входят ли они в индексы, есть ли HOT updates в `pg_stat_user_tables`, и иногда снижаю `fillfactor`, чтобы уменьшить bloat и нагрузку на индексы.

## Snapshots и видимость данных

Snapshot - это представление транзакции о том, какие transaction IDs уже committed, какие еще active, а какие future.

Упрощенно tuple видима, если:

- транзакция из `xmin` committed;
- транзакция из `xmax` не committed или tuple не была удалена для данного snapshot-а;
- tuple не создана будущей или незавершенной транзакцией.

Пример:

```sql
-- Session A
BEGIN;
SELECT balance FROM accounts WHERE id = 1;

-- Session B
BEGIN;
UPDATE accounts SET balance = balance + 50 WHERE id = 1;
COMMIT;

-- Session A
SELECT balance FROM accounts WHERE id = 1;
COMMIT;
```

Результат во второй выборке Session A зависит от isolation level.

## Isolation levels

PostgreSQL поддерживает стандартные уровни изоляции, но с особенностями реализации.

### Read Committed

Уровень по умолчанию.

- Каждый SQL statement получает новый snapshot.
- Повторный `SELECT` в той же транзакции может увидеть новые committed данные.
- Не защищает от non-repeatable reads и phantom reads в общем смысле.

Пример:

```sql
BEGIN ISOLATION LEVEL READ COMMITTED;
SELECT balance FROM accounts WHERE id = 1;
-- другая транзакция сделала COMMIT
SELECT balance FROM accounts WHERE id = 1;
COMMIT;
```

### Repeatable Read

- Snapshot фиксируется на начало транзакции.
- Повторные чтения видят одну и ту же картину.
- В PostgreSQL это snapshot isolation.
- Возможны serialization anomalies, поэтому для строгой сериализуемости нужен `SERIALIZABLE`.

### Serializable

- Самый строгий уровень.
- PostgreSQL использует SSI, Serializable Snapshot Isolation.
- Может завершать транзакции ошибкой `could not serialize access due to read/write dependencies`.
- Приложение должно уметь retry-ить транзакцию.

Senior answer:

> В PostgreSQL `SERIALIZABLE` не означает, что все грубо блокируется. Это optimistic-подход на базе snapshot isolation и predicate conflict detection. Поэтому корректный PHP-код должен иметь retry policy для serialization failures.

## Типичные аномалии

### Lost update

Плохой паттерн:

```php
$balance = $repo->getBalance($accountId);
$repo->setBalance($accountId, $balance + 100);
```

Лучше:

```sql
UPDATE accounts
SET balance = balance + 100
WHERE id = 1;
```

Или использовать row lock:

```sql
BEGIN;
SELECT balance FROM accounts WHERE id = 1 FOR UPDATE;
UPDATE accounts SET balance = balance + 100 WHERE id = 1;
COMMIT;
```

### Write skew

Две транзакции читают общий инвариант и обновляют разные строки. `REPEATABLE READ` может не защитить, `SERIALIZABLE` может обнаружить конфликт.

## Locks в PostgreSQL

MVCC не отменяет locks. Блокировки нужны для защиты структур данных, конфликтующих writes, DDL и явной синхронизации.

### Row-level locks

Используются при `UPDATE`, `DELETE`, `SELECT ... FOR UPDATE`.

```sql
SELECT * FROM jobs
WHERE status = 'pending'
ORDER BY id
LIMIT 10
FOR UPDATE SKIP LOCKED;
```

Паттерн `SKIP LOCKED` полезен для очередей, но важно понимать: он пропускает заблокированные строки и не гарантирует строгую справедливость.

### Table-level locks

Некоторые операции берут locks на таблицу. Например, `ALTER TABLE` часто требует сильные блокировки и может заблокировать production traffic.

Проверка блокировок:

```sql
SELECT
    blocked.pid AS blocked_pid,
    blocked.query AS blocked_query,
    blocking.pid AS blocking_pid,
    blocking.query AS blocking_query
FROM pg_stat_activity blocked
JOIN pg_locks blocked_locks ON blocked_locks.pid = blocked.pid
JOIN pg_locks blocking_locks
    ON blocking_locks.locktype = blocked_locks.locktype
   AND blocking_locks.database IS NOT DISTINCT FROM blocked_locks.database
   AND blocking_locks.relation IS NOT DISTINCT FROM blocked_locks.relation
   AND blocking_locks.page IS NOT DISTINCT FROM blocked_locks.page
   AND blocking_locks.tuple IS NOT DISTINCT FROM blocked_locks.tuple
   AND blocking_locks.transactionid IS NOT DISTINCT FROM blocked_locks.transactionid
   AND blocking_locks.classid IS NOT DISTINCT FROM blocked_locks.classid
   AND blocking_locks.objid IS NOT DISTINCT FROM blocked_locks.objid
   AND blocking_locks.objsubid IS NOT DISTINCT FROM blocked_locks.objsubid
   AND blocking_locks.pid != blocked_locks.pid
JOIN pg_stat_activity blocking ON blocking.pid = blocking_locks.pid
WHERE NOT blocked_locks.granted
  AND blocking_locks.granted;
```

### Advisory locks

Приложенческие блокировки, полезны для distributed cron, миграций, single-flight задач.

```sql
SELECT pg_try_advisory_lock(12345);
SELECT pg_advisory_unlock(12345);
```

Pitfall: advisory lock держится на session или transaction в зависимости от функции. В PHP с persistent connections или pool-ом это особенно важно.

## VACUUM

VACUUM очищает dead tuples, обновляет visibility map, помогает index-only scans и предотвращает transaction ID wraparound.

Обычный `VACUUM`:

- не возвращает место операционной системе в большинстве случаев;
- делает место повторно используемым внутри таблицы;
- работает конкурентно с обычными запросами;
- может чистить индексы от ссылок на dead tuples.

`VACUUM FULL`:

- переписывает таблицу;
- возвращает место ОС;
- требует сильную блокировку;
- обычно опасен для production без окна обслуживания.

Senior answer:

> VACUUM не "сжимает таблицу" в обычном смысле. Он помечает dead tuples как reusable space. Если нужен физический shrink, это уже `VACUUM FULL`, `CLUSTER`, `pg_repack` или пересоздание таблицы, но это отдельный operational risk.

## Autovacuum

Autovacuum автоматически запускает VACUUM и ANALYZE.

Основные причины запуска:

- накопилось много dead tuples;
- изменилось достаточно строк для обновления статистики;
- таблица приближается к wraparound risk.

Важные настройки:

```text
autovacuum = on
autovacuum_max_workers
autovacuum_naptime
autovacuum_vacuum_threshold
autovacuum_vacuum_scale_factor
autovacuum_analyze_threshold
autovacuum_analyze_scale_factor
autovacuum_vacuum_cost_limit
autovacuum_vacuum_cost_delay
```

Для больших таблиц дефолтный `scale_factor` часто слишком высокий.

Пример per-table настройки:

```sql
ALTER TABLE events SET (
    autovacuum_vacuum_scale_factor = 0.02,
    autovacuum_analyze_scale_factor = 0.01,
    autovacuum_vacuum_cost_limit = 2000
);
```

Мониторинг:

```sql
SELECT
    relname,
    n_live_tup,
    n_dead_tup,
    last_vacuum,
    last_autovacuum,
    last_analyze,
    last_autoanalyze,
    vacuum_count,
    autovacuum_count
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC
LIMIT 20;
```

## Bloat

Bloat - лишнее физическое место в таблицах и индексах из-за dead tuples, page splits, frequent updates/deletes, неэффективного VACUUM.

Симптомы:

- таблица сильно больше ожидаемого объема данных;
- sequential scans читают много лишних страниц;
- индексы растут, но cardinality не растет;
- `VACUUM` не успевает;
- long-running transactions удерживают старые tuple versions.

Причины:

- долгие транзакции;
- idle in transaction;
- массовые `UPDATE`/`DELETE`;
- часто обновляемые indexed columns;
- слишком высокий `autovacuum_vacuum_scale_factor`;
- мало autovacuum workers;
- replication slot удерживает WAL и старые горизонты очистки.

Диагностика долгих транзакций:

```sql
SELECT
    pid,
    state,
    now() - xact_start AS xact_age,
    now() - query_start AS query_age,
    wait_event_type,
    wait_event,
    query
FROM pg_stat_activity
WHERE xact_start IS NOT NULL
ORDER BY xact_start;
```

## Freeze и transaction ID wraparound

PostgreSQL использует transaction IDs ограниченного размера. Старые tuple должны быть frozen, чтобы система не перепутала очень старые транзакции с будущими.

Что важно знать:

- VACUUM freeze помечает старые tuples как видимые всем будущим транзакциям.
- Если база приближается к wraparound, PostgreSQL принудительно запускает anti-wraparound vacuum.
- Игнорирование freeze может привести к emergency shutdown, чтобы защитить данные.

Проверка возраста:

```sql
SELECT
    datname,
    age(datfrozenxid) AS xid_age
FROM pg_database
ORDER BY xid_age DESC;
```

## WAL

WAL, Write-Ahead Logging, это журнал изменений. Перед тем как грязные страницы данных будут записаны на диск, соответствующие WAL records должны быть надежно записаны.

Зачем нужен WAL:

- crash recovery;
- physical replication;
- point-in-time recovery;
- logical decoding;
- durability гарантия после commit.

Упрощенный commit path:

1. Транзакция меняет страницы в shared buffers.
2. PostgreSQL пишет WAL records.
3. При commit WAL должен быть flushed в соответствии с `synchronous_commit`.
4. Позже checkpoint/background writer сбрасывают dirty pages на диск.

Senior answer:

> PostgreSQL не обязан сразу писать измененные data pages на диск при commit. Durability обеспечивается тем, что WAL записан раньше data pages. После crash PostgreSQL replay-ит WAL и восстанавливает консистентное состояние.

## Checkpoints

Checkpoint - момент, после которого recovery может стартовать с известной точки, потому что dirty pages до этой точки постепенно сбрасываются на диск.

Важные параметры:

```text
checkpoint_timeout
max_wal_size
min_wal_size
checkpoint_completion_target
```

Проблемы:

- слишком частые checkpoints создают write spikes;
- слишком большой WAL увеличивает время recovery;
- плохая настройка может давать latency spikes на write-heavy нагрузке.

Признак в логах:

```text
checkpoints are occurring too frequently
```

Что делать:

- увеличить `max_wal_size`;
- проверить write workload;
- смотреть `pg_stat_bgwriter` и `pg_stat_checkpointer` в новых версиях;
- не лечить симптом без понимания I/O.

## Replication basics

### Physical streaming replication

Replica получает WAL и replay-ит его. Реплика физически совместима с primary.

Плюсы:

- простая HA/read replica модель;
- точная копия primary;
- подходит для failover.

Минусы:

- нельзя реплицировать только часть таблиц;
- major version и storage-level совместимость важны;
- read queries на replica могут конфликтовать с WAL replay.

### Logical replication

Реплицирует логические изменения: `INSERT`, `UPDATE`, `DELETE`.

Плюсы:

- можно реплицировать отдельные таблицы;
- полезно для миграций, интеграций, zero-downtime переходов;
- может работать между разными major versions при соблюдении ограничений.

Минусы:

- не все DDL реплицируется автоматически;
- нужен primary key или replica identity для updates/deletes;
- replication slots могут удерживать WAL.

Replication lag:

```sql
SELECT
    application_name,
    state,
    sync_state,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn,
    write_lag,
    flush_lag,
    replay_lag
FROM pg_stat_replication;
```

PHP pitfall:

> Если приложение читает с replica сразу после записи в primary, возможна read-after-write inconsistency. Для критичных сценариев нужен read-your-writes: читать с primary, использовать session consistency, LSN tracking или другой механизм.

## Query planner

Planner выбирает план выполнения запроса на основе статистики, стоимости операций, доступных индексов, constraints, параметров и SQL-структуры.

Он оценивает:

- cardinality: сколько строк вернет узел;
- selectivity: насколько фильтр отсекает данные;
- cost: условная стоимость запуска и выполнения;
- join order;
- join algorithm;
- access path: seq scan, index scan, bitmap scan и т.д.

Planner не знает фактические данные идеально. Он работает по статистике.

## Statistics

`ANALYZE` собирает статистику:

- approximate row count;
- most common values;
- histogram;
- null fraction;
- correlation;
- distinct estimate.

Если статистика устарела, planner ошибается.

Пример:

```sql
ANALYZE orders;

SELECT
    schemaname,
    tablename,
    attname,
    n_distinct,
    most_common_vals,
    most_common_freqs,
    histogram_bounds,
    correlation
FROM pg_stats
WHERE tablename = 'orders';
```

Для skewed data может потребоваться увеличить statistics target:

```sql
ALTER TABLE orders ALTER COLUMN status SET STATISTICS 1000;
ANALYZE orders;
```

Extended statistics помогают при коррелированных колонках:

```sql
CREATE STATISTICS orders_customer_status_stats
ON customer_id, status
FROM orders;

ANALYZE orders;
```

Senior answer:

> Если `EXPLAIN ANALYZE` показывает, что estimated rows отличаются от actual rows на порядки, я сначала думаю о статистике, корреляции колонок, skewed data, expressions, partial indexes и параметризованных запросах, а не сразу добавляю индекс.

## Join algorithms

### Nested Loop

Хорош для малого outer set и быстрого lookup по inner side.

Pitfall: если planner ошибся и outer set большой, nested loop может стать катастрофой.

### Hash Join

Строит hash table по одной стороне join-а. Хорош для equality joins и больших наборов.

Pitfall: если hash не помещается в `work_mem`, будут batches и disk spill.

### Merge Join

Требует отсортированные входы или сортировку. Может быть хорош, если данные уже идут по index order.

## Index internals overview

### B-tree

Основной индекс по умолчанию.

Подходит для:

- equality;
- range queries;
- ordering;
- prefix composite index;
- unique constraints.

Composite index rule:

```sql
CREATE INDEX idx_orders_customer_status_created
ON orders (customer_id, status, created_at DESC);
```

Эффективен для:

```sql
WHERE customer_id = ?
WHERE customer_id = ? AND status = ?
WHERE customer_id = ? AND status = ? ORDER BY created_at DESC
```

Менее полезен для:

```sql
WHERE status = ?
```

### GIN

Подходит для many-to-many внутри значения:

- arrays;
- `jsonb` containment;
- full-text search;
- trigrams через `pg_trgm`.

Пример:

```sql
CREATE INDEX idx_events_payload_gin
ON events USING gin (payload jsonb_path_ops);
```

### GiST

Обобщенный индексный метод. Часто используется для геометрии, ranges, full-text search, exclusion constraints.

### BRIN

Block Range Index. Маленький индекс, полезен для огромных append-only таблиц с физической корреляцией данных.

Пример:

```sql
CREATE INDEX idx_logs_created_brin
ON logs USING brin (created_at);
```

Хорош для event/log таблиц, где `created_at` растет вместе с физическим порядком вставки.

### Hash

Индекс для equality. В современных версиях WAL-logged, но B-tree обычно универсальнее.

## Index-only scan и visibility map

Index-only scan возможен, когда PostgreSQL может получить данные из индекса и не ходить в heap. Но ему нужно понимать, что tuple видима всем. Для этого используется visibility map.

Если таблица плохо vacuum-ится, index-only scan может деградировать в heap fetches.

Пример проверки:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, created_at
FROM orders
WHERE customer_id = 123
ORDER BY created_at DESC
LIMIT 20;
```

Смотреть:

- `Index Only Scan`;
- `Heap Fetches`;
- shared hit/read buffers.

## EXPLAIN

`EXPLAIN` показывает план без выполнения запроса.

```sql
EXPLAIN
SELECT * FROM orders WHERE customer_id = 123;
```

`EXPLAIN ANALYZE` выполняет запрос и показывает фактические метрики.

```sql
EXPLAIN (ANALYZE, BUFFERS, VERBOSE)
SELECT * FROM orders WHERE customer_id = 123;
```

Важно: `EXPLAIN ANALYZE` реально выполняет запрос. Для `INSERT`, `UPDATE`, `DELETE` используйте транзакцию с rollback, если нужно безопасно посмотреть план.

```sql
BEGIN;
EXPLAIN (ANALYZE, BUFFERS)
UPDATE orders SET status = 'archived' WHERE created_at < now() - interval '1 year';
ROLLBACK;
```

## Как читать EXPLAIN ANALYZE

Смотреть сверху вниз для общей структуры, но время часто анализировать снизу вверх.

Ключевые поля:

- `cost=startup..total` - оценочная стоимость.
- `rows` - оценка количества строк.
- `width` - оценка размера строки.
- `actual time=start..end` - фактическое время.
- `actual rows` - фактическое количество строк.
- `loops` - сколько раз узел выполнялся.
- `Buffers: shared hit/read/dirtied/written` - работа с буферами.
- `Rows Removed by Filter` - сколько строк прочитали и отфильтровали.
- `Sort Method` - сортировка в памяти или на диске.
- `Heap Fetches` - обращения в heap при index-only scan.

Пример проблемы:

```text
Nested Loop  (cost=0.42..100.00 rows=10 width=64)
             (actual time=0.100..9500.000 rows=500000 loops=1)
```

Интерпретация: planner ожидал 10 строк, получил 500000. Надо смотреть статистику, фильтры, join conditions, параметры, корреляции.

## Seq Scan не всегда плохо

Senior должен сказать:

> Sequential scan не является ошибкой сам по себе. Если запрос читает большую долю таблицы или таблица маленькая, seq scan может быть дешевле index scan. Проблема не в самом `Seq Scan`, а в несоответствии плана задаче и фактической стоимости.

## Bitmap Scan

Bitmap Index Scan + Bitmap Heap Scan часто используется, когда нужно прочитать много строк через индекс, но random access по одной строке был бы дорогим.

Сигналы:

- `Recheck Cond`;
- `Heap Blocks: exact/lossy`;
- lossy blocks могут означать нехватку `work_mem`.

## Slow query diagnostics

Базовый порядок диагностики:

1. Найти запрос: `pg_stat_statements`, logs, APM, slow query log.
2. Получить фактический план: `EXPLAIN (ANALYZE, BUFFERS)` на похожих параметрах.
3. Сравнить estimated rows и actual rows.
4. Проверить, где время: scan, join, sort, aggregate, lock wait, I/O.
5. Проверить статистику и `ANALYZE`.
6. Проверить индексы: есть ли подходящий, используется ли, не мешают ли casts/functions.
7. Проверить bloat и autovacuum.
8. Проверить locks и долгие транзакции.
9. Проверить ORM-паттерн: N+1, лишние joins, offset pagination, `SELECT *`.
10. Внести минимальное изменение: индекс, rewrite запроса, batch size, pagination, stats, config.

`pg_stat_statements`:

```sql
SELECT
    query,
    calls,
    total_exec_time,
    mean_exec_time,
    rows,
    shared_blks_hit,
    shared_blks_read,
    temp_blks_read,
    temp_blks_written
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;
```

## Частые production issues

### Idle in transaction

Проблема:

- держит snapshot;
- мешает VACUUM чистить dead tuples;
- может держать locks;
- вызывает bloat.

PHP-причины:

- забыли `commit`/`rollback`;
- транзакция открыта вокруг HTTP/API вызова;
- долгое выполнение бизнес-логики внутри транзакции;
- exception path не закрывает транзакцию.

Решение:

- короткие транзакции;
- `idle_in_transaction_session_timeout`;
- правильный transaction middleware;
- не делать network I/O внутри DB transaction.

### Offset pagination

```sql
SELECT * FROM orders
ORDER BY id
LIMIT 50 OFFSET 1000000;
```

PostgreSQL должен пройти и отбросить много строк. Лучше keyset pagination:

```sql
SELECT * FROM orders
WHERE id > :last_id
ORDER BY id
LIMIT 50;
```

### Functions and casts break index usage

Плохо:

```sql
WHERE date(created_at) = '2026-06-29'
```

Лучше:

```sql
WHERE created_at >= '2026-06-29'::date
  AND created_at <  '2026-06-30'::date
```

Или expression index, если это осознанная модель:

```sql
CREATE INDEX idx_orders_created_date
ON orders ((date(created_at)));
```

### Low selectivity index

Индекс по boolean/status с двумя значениями часто бесполезен сам по себе. Лучше partial index:

```sql
CREATE INDEX idx_jobs_pending_created
ON jobs (created_at)
WHERE status = 'pending';
```

### N+1 из ORM

Симптом:

- много одинаковых query fingerprints;
- маленькое mean time, но большое total time;
- высокая нагрузка на connections и CPU.

Решение:

- eager loading;
- batch loading;
- aggregation в SQL;
- ограничение сериализации сущностей.

### Too many connections

PostgreSQL connection - не легковесный HTTP request. Большое число connections увеличивает память и contention.

Решения:

- PgBouncer;
- ограничение PHP-FPM workers;
- настройка pool size;
- короткие транзакции;
- server-side timeouts.

### Lock timeout during migrations

DDL может ждать lock и блокировать последующие запросы.

Практики:

```sql
SET lock_timeout = '5s';
SET statement_timeout = '5min';
```

- использовать `CREATE INDEX CONCURRENTLY`;
- не добавлять `NOT NULL DEFAULT` без понимания версии и rewrite behavior;
- разделять миграцию схемы и backfill;
- backfill делать батчами;
- проверять locks перед миграцией.

## Senior answers: короткие формулировки

### Почему PostgreSQL раздувается после UPDATE?

`UPDATE` создает новую версию строки, старая версия остается для старых snapshots. VACUUM потом освобождает dead tuples для повторного использования. Если есть долгие транзакции или autovacuum не успевает, dead tuples копятся и появляется bloat.

### Почему индекс есть, но не используется?

Возможные причины: planner считает seq scan дешевле, низкая selectivity, устаревшая статистика, маленькая таблица, expression/cast не совпадает с индексом, неправильный порядок колонок composite index, параметризованный запрос, partial index не покрывает predicate, collation/operator class mismatch.

### Чем `VACUUM` отличается от `ANALYZE`?

`VACUUM` чистит dead tuples и обслуживает storage/visibility/freeze. `ANALYZE` собирает статистику для planner-а. Autovacuum обычно делает и то и другое, но это разные задачи.

### Почему `EXPLAIN ANALYZE` опасен?

Он реально выполняет запрос. Для write-запросов это изменит данные, если не обернуть в транзакцию и `ROLLBACK`. Для тяжелых read-запросов может создать production-нагрузку.

### Что такое WAL простыми словами?

Это журнал изменений, который записывается перед измененными data pages. Он нужен для crash recovery, replication и point-in-time recovery. Commit durable, когда нужные WAL records сброшены согласно настройкам durability.

### Что делать с replication lag?

Сначала понять тип лага: network write, flush на replica, replay, long query conflict, I/O saturation. Потом смотреть `pg_stat_replication`, нагрузку replica, WAL generation rate, long-running queries, slots. В приложении не читать критичные read-after-write данные с отстающей replica.

## Мини-практика

### Практика 1: найти проблему в плане

Дано:

```text
Index Scan using idx_orders_status on orders
  (cost=0.42..500.00 rows=100 width=120)
  (actual time=0.050..12000.000 rows=2500000 loops=1)
  Index Cond: (status = 'paid')
```

Что сказать:

- статус слишком низко селективен;
- planner ошибся в cardinality или индекс реально невыгоден;
- надо смотреть распределение `status`, актуальность `ANALYZE`, partial/composite index, условия запроса;
- возможно seq scan или partitioning лучше;
- если нужны последние paid orders, индекс `(status, created_at DESC)` или partial index может помочь.

### Практика 2: очередь задач

Задача: несколько воркеров должны безопасно брать jobs.

```sql
WITH picked AS (
    SELECT id
    FROM jobs
    WHERE status = 'pending'
    ORDER BY id
    LIMIT 10
    FOR UPDATE SKIP LOCKED
)
UPDATE jobs
SET status = 'processing', started_at = now()
WHERE id IN (SELECT id FROM picked)
RETURNING *;
```

Что обсудить:

- нужен индекс по pending jobs;
- транзакция должна быть короткой;
- зависшие `processing` jobs нужно переоткладывать;
- `SKIP LOCKED` может давать starvation;
- для большой очереди лучше partial index.

### Практика 3: безопасный backfill

Плохой вариант:

```sql
UPDATE users SET normalized_email = lower(email);
```

Лучше:

- добавить nullable колонку;
- обновлять батчами по primary key;
- ограничить batch size;
- делать паузы;
- мониторить locks, lag, dead tuples;
- после backfill добавить constraint/index concurrently, где возможно.

## Self-check

- Почему `UPDATE` в PostgreSQL может увеличивать размер таблицы?
- Чем snapshot в `READ COMMITTED` отличается от `REPEATABLE READ`?
- Почему долгие транзакции мешают VACUUM?
- Что делает `VACUUM`, а что делает `VACUUM FULL`?
- Зачем нужен freeze?
- Что гарантирует WAL?
- Почему checkpoints могут вызывать latency spikes?
- Чем physical replication отличается от logical replication?
- Что означает большая разница между estimated rows и actual rows?
- Когда seq scan лучше index scan?
- Почему `EXPLAIN ANALYZE` может быть опасен?
- Как диагностировать lock contention?
- Какой индекс выбрать для `WHERE status = 'pending' ORDER BY created_at LIMIT 100`?
- Почему `WHERE date(created_at) = ?` может не использовать обычный индекс по `created_at`?
- Как PHP-приложение может вызвать `idle in transaction`?

## Checklist перед интервью

- Умею объяснить MVCC через tuple versions, `xmin`, `xmax`, snapshots.
- Умею объяснить, почему читатели не блокируют писателей.
- Знаю отличия `READ COMMITTED`, `REPEATABLE READ`, `SERIALIZABLE`.
- Знаю, когда нужен `SELECT FOR UPDATE`, `SKIP LOCKED`, advisory locks.
- Умею объяснить VACUUM, autovacuum, bloat, freeze.
- Понимаю WAL, checkpoints, crash recovery и replication.
- Умею читать `EXPLAIN (ANALYZE, BUFFERS)`.
- Сравниваю estimated rows и actual rows.
- Не считаю seq scan автоматически плохим.
- Знаю базовые index types: B-tree, GIN, GiST, BRIN.
- Умею диагностировать slow query через `pg_stat_statements` и план.
- Помню PHP-specific риски: N+1, long transactions, too many connections, read replica lag.
- Могу предложить безопасную production-миграцию без долгих locks.

## Ссылки

- PostgreSQL Documentation: MVCC - https://www.postgresql.org/docs/current/mvcc.html
- PostgreSQL Documentation: Transaction Isolation - https://www.postgresql.org/docs/current/transaction-iso.html
- PostgreSQL Documentation: Explicit Locking - https://www.postgresql.org/docs/current/explicit-locking.html
- PostgreSQL Documentation: Routine Vacuuming - https://www.postgresql.org/docs/current/routine-vacuuming.html
- PostgreSQL Documentation: Autovacuum - https://www.postgresql.org/docs/current/runtime-config-autovacuum.html
- PostgreSQL Documentation: WAL - https://www.postgresql.org/docs/current/wal.html
- PostgreSQL Documentation: Checkpoints - https://www.postgresql.org/docs/current/wal-configuration.html
- PostgreSQL Documentation: High Availability, Load Balancing, and Replication - https://www.postgresql.org/docs/current/high-availability.html
- PostgreSQL Documentation: Using EXPLAIN - https://www.postgresql.org/docs/current/using-explain.html
- PostgreSQL Documentation: Planner Statistics - https://www.postgresql.org/docs/current/planner-stats.html
- PostgreSQL Documentation: Indexes - https://www.postgresql.org/docs/current/indexes.html
- PostgreSQL Wiki: VACUUM FULL - https://wiki.postgresql.org/wiki/VACUUM_FULL
- PostgreSQL Wiki: Index Maintenance - https://wiki.postgresql.org/wiki/Index_Maintenance
- PostgreSQL Wiki: Don't Do This - https://wiki.postgresql.org/wiki/Don%27t_Do_This
- Use The Index, Luke: PostgreSQL indexing and query plans - https://use-the-index-luke.com/
- pganalyze Blog: PostgreSQL VACUUM and bloat articles - https://pganalyze.com/blog
- Cybertec Blog: PostgreSQL performance, VACUUM, locks - https://www.cybertec-postgresql.com/en/blog/
- 2ndQuadrant/EDB Blog: PostgreSQL internals and performance - https://www.enterprisedb.com/blog
