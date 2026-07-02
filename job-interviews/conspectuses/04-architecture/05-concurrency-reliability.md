# Concurrency and Reliability: Locks, Idempotency, Outbox, Inbox and Sagas

<!-- markdownlint-disable MD013 -->

Конспект для senior/lead backend interviews. Фокус: как проектировать систему, которая корректно работает при параллельных запросах, сбоях сети, retries, падениях воркеров, дублях сообщений и частичной недоступности зависимостей.

## Что проверяют на интервью

Интервьюер обычно хочет понять не знание названий паттернов, а способность рассуждать о correctness under failure.

Хороший senior-ответ почти всегда содержит:

- где находится source of truth;
- какие инварианты нельзя нарушить;
- какая consistency model допустима;
- какие операции идемпотентны, а какие нет;
- где нужны transactions и locks;
- что произойдет при timeout, retry, падении процесса и duplicate message;
- как система восстанавливается;
- какие metrics и alerts нужны.

## Concurrency Basics

Concurrency - несколько задач логически выполняются в перекрывающиеся периоды времени. Parallelism - физическое выполнение одновременно.

В PHP backend concurrency обычно появляется не из потоков внутри одного процесса, а из:

- нескольких PHP-FPM workers;
- нескольких queue workers;
- нескольких RabbitMQ consumers;
- нескольких cron/job процессов;
- горизонтального масштабирования приложения;
- повторных HTTP-запросов клиента;
- retries от load balancer, клиента, broker или scheduler.

## Race Conditions

Race condition - ошибка, при которой результат зависит от непредсказуемого порядка параллельных операций.

Классический пример списания остатка:

```php
$account = Account::find($id);

if ($account->balance >= 100) {
    $account->balance -= 100;
    $account->save();
}
```

Если два запроса одновременно прочитали `balance = 100`, оба могут пройти проверку и сохранить списание. Инвариант `balance >= 0` нарушен.

### Critical Section

Critical section - участок кода, где выполняется чтение/проверка/изменение общего состояния и где нельзя допустить неконтролируемое параллельное выполнение.

Важно: critical section должна быть как можно меньше. Нельзя держать lock во время HTTP-вызова внешнего сервиса, отправки email или долгой CPU-операции, если это можно вынести за пределы transaction.

## Типовые гонки в Backend

### Lost Update

Два процесса читают одно состояние, оба изменяют, последний `save()` затирает результат первого.

Решения:

- atomic SQL `UPDATE ... SET value = value + 1`;
- optimistic locking через `version`;
- pessimistic lock `SELECT ... FOR UPDATE`;
- constraint и повтор transaction.

### Check-Then-Act

Код сначала проверяет, потом действует, но между проверкой и действием состояние меняется.

Плохой пример:

```php
if (!Order::where('external_id', $externalId)->exists()) {
    Order::create(['external_id' => $externalId]);
}
```

Правильнее:

```sql
CREATE UNIQUE INDEX orders_external_id_unique ON orders (external_id);
```

```php
try {
    Order::create(['external_id' => $externalId]);
} catch (UniqueConstraintViolationException $e) {
    // Уже создано параллельным запросом. Дальше читаем существующую запись.
}
```

### Double Submit

Пользователь дважды нажал кнопку оплаты, мобильное приложение повторило запрос после timeout, gateway отправил webhook два раза.

Решения:

- idempotency key;
- unique constraint на business key;
- status model;
- inbox/deduplication для входящих events.

### Duplicate Message Processing

Broker доставил одно сообщение несколько раз, consumer упал после обработки, но до ack.

Решения:

- handler должен быть idempotent;
- inbox table с `message_id`;
- unique constraints;
- ack только после commit.

## Optimistic Locking

Optimistic locking предполагает, что конфликты редки. Мы не блокируем строку при чтении, но при записи проверяем, что version не изменилась.

PostgreSQL пример:

```sql
ALTER TABLE orders ADD COLUMN version integer NOT NULL DEFAULT 1;
```

```php
$affected = DB::update(
    'UPDATE orders
     SET status = ?, version = version + 1
     WHERE id = ? AND version = ?',
    ['paid', $orderId, $expectedVersion]
);

if ($affected === 0) {
    throw new ConcurrentModificationException('Order was changed concurrently');
}
```

Когда подходит:

- конфликты редки;
- пользователь может повторить действие;
- важна высокая throughput;
- можно показать ошибку `record was changed, reload and retry`;
- можно безопасно повторить command.

Ловушки:

- забыли включить `version` в `WHERE`;
- обновляют через ORM `save()` без проверки version;
- делают несколько связанных изменений, но version только у одной entity;
- retry без ограничения превращается в storm;
- конфликт скрывают и перезаписывают чужие изменения.

## Pessimistic Locking

Pessimistic locking предполагает, что конфликт вероятен или цена ошибки высока. Мы явно блокируем строки до конца transaction.

Laravel пример:

```php
DB::transaction(function () use ($accountId, $amount): void {
    $account = Account::query()
        ->whereKey($accountId)
        ->lockForUpdate()
        ->firstOrFail();

    if ($account->balance < $amount) {
        throw new InsufficientFundsException();
    }

    $account->balance -= $amount;
    $account->save();
});
```

Symfony/Doctrine пример:

```php
$entityManager->wrapInTransaction(function () use ($entityManager, $accountId, $amount): void {
    $account = $entityManager->find(
        Account::class,
        $accountId,
        LockMode::PESSIMISTIC_WRITE,
    );

    if ($account->balance() < $amount) {
        throw new InsufficientFundsException();
    }

    $account->withdraw($amount);
});
```

Когда подходит:

- деньги, остатки, лимиты, квоты;
- нельзя допустить нарушение инварианта;
- critical section короткая;
- конфликт ожидаем;
- проще подождать lock, чем обрабатывать version conflict.

Ловушки:

- lock работает только внутри transaction;
- ORM lazy loading может неявно расширить transaction;
- долгие операции внутри transaction приводят к lock contention;
- разный порядок locks приводит к deadlock;
- `SELECT` без `FOR UPDATE` не защищает от параллельного изменения;
- row lock не защищает от вставки новой строки, если инвариант относится к диапазону.

## PostgreSQL Locks

### Row-Level Locks

Основные варианты:

- `FOR UPDATE` - блокирует строки для изменения;
- `FOR NO KEY UPDATE` - слабее, если не меняется key;
- `FOR SHARE` - shared lock для чтения;
- `FOR KEY SHARE` - защита foreign keys.

```sql
BEGIN;

SELECT *
FROM accounts
WHERE id = 42
FOR UPDATE;

UPDATE accounts
SET balance = balance - 100
WHERE id = 42;

COMMIT;
```

### NOWAIT and SKIP LOCKED

`NOWAIT` сразу падает, если строка заблокирована.

```sql
SELECT * FROM jobs
WHERE id = 10
FOR UPDATE NOWAIT;
```

`SKIP LOCKED` полезен для конкурентного разбора очереди из таблицы.

```sql
SELECT id
FROM outbox_messages
WHERE processed_at IS NULL
ORDER BY id
FOR UPDATE SKIP LOCKED
LIMIT 100;
```

Ловушка: `SKIP LOCKED` может приводить к starvation отдельных записей, если они постоянно заблокированы или падают при обработке.

### Advisory Locks

PostgreSQL advisory locks - пользовательские locks по числовому ключу.

```sql
SELECT pg_try_advisory_lock(12345);
SELECT pg_advisory_unlock(12345);
```

Transaction-level вариант безопаснее для большинства backend-сценариев:

```sql
SELECT pg_try_advisory_xact_lock(12345);
```

Когда использовать:

- защита cron/job от параллельного запуска;
- lock по business key, когда строки еще нет;
- coarse-grained lock для редких операций.

Ловушки:

- надо стабильно хешировать business key;
- session-level lock можно забыть освободить;
- advisory lock не заменяет constraints;
- не видно напрямую на уровне domain model.

### Isolation Levels

PostgreSQL по умолчанию использует `READ COMMITTED`. Это нормально для большинства операций, если инварианты защищены constraints, row locks или atomic update.

`REPEATABLE READ` дает стабильный snapshot, но не решает все business invariants автоматически.

`SERIALIZABLE` может предотвращать более широкий класс anomalies, но требует обработки serialization failure и retry transaction.

Senior-ответ: isolation level - не магия. Надо понимать конкретный invariant, тип anomaly и цену retry.

## Atomic Operations and Constraints

Самая надежная блокировка часто не выглядит как lock в коде.

### Atomic UPDATE

```sql
UPDATE accounts
SET balance = balance - :amount
WHERE id = :id
  AND balance >= :amount;
```

Если `rowCount() === 0`, средств не хватило или аккаунта нет.

Плюсы:

- одна SQL-операция;
- меньше времени удержания lock;
- хорошо масштабируется;
- invariant проверяется в БД.

### Unique Constraint

```sql
CREATE UNIQUE INDEX payments_idempotency_key_unique
ON payments (merchant_id, idempotency_key);
```

Это лучше, чем `exists()` перед `insert()`.

### Partial Unique Index

Например, только одна активная subscription на пользователя.

```sql
CREATE UNIQUE INDEX subscriptions_one_active_per_user
ON subscriptions (user_id)
WHERE status = 'active';
```

## Distributed Locks

Distributed lock нужен, когда несколько процессов на разных машинах должны эксклюзивно выполнить действие, а обычной DB transaction недостаточно или ресурс не находится в одной БД.

Примеры:

- один scheduler job на весь cluster;
- один reindex на tenant;
- защита внешнего API от параллельных несовместимых вызовов;
- leader election для легковесного процесса.

### Redis Lock

Минимальный корректный вариант:

```php
$token = bin2hex(random_bytes(16));
$ttlMs = 30000;

$acquired = $redis->set('lock:tenant:42:reindex', $token, ['NX', 'PX' => $ttlMs]);

if (!$acquired) {
    throw new LockNotAcquiredException();
}

try {
    reindexTenant(42);
} finally {
    $script = <<<'LUA'
if redis.call('get', KEYS[1]) == ARGV[1] then
    return redis.call('del', KEYS[1])
end
return 0
LUA;

    $redis->eval($script, ['lock:tenant:42:reindex', $token], 1);
}
```

Ключевые детали:

- `NX` - установить только если ключа нет;
- TTL обязателен, иначе lock может остаться навсегда;
- token обязателен, чтобы не удалить чужой lock;
- unlock должен быть атомарным Lua script;
- TTL должен быть больше ожидаемой critical section с запасом.

### Laravel Atomic Locks

```php
Cache::lock('tenant:42:reindex', 30)->block(5, function (): void {
    reindexTenant(42);
});
```

Важно проверить cache driver. Для distributed lock нужны Redis, Memcached, DynamoDB или database driver с понятной семантикой. `array` и локальный file cache не подходят для нескольких instances.

### Symfony Lock

```php
$lock = $lockFactory->createLock('tenant:42:reindex', 30);

if (!$lock->acquire()) {
    throw new LockNotAcquiredException();
}

try {
    reindexTenant(42);
} finally {
    $lock->release();
}
```

Symfony Lock поддерживает Redis, PDO, PostgreSQL advisory locks, Semaphore и другие stores.

### Redlock Caveats

Redlock - алгоритм distributed lock поверх нескольких независимых Redis instances. Вокруг него есть известная дискуссия: antirez предложил алгоритм, Martin Kleppmann критиковал его для correctness-critical систем.

Senior-позиция:

- Redis lock можно использовать для efficiency locks: не запускать лишнюю работу, уменьшить дубли, защитить дорогую операцию;
- нельзя безоговорочно полагаться на Redis lock для correctness-critical invariants вроде денег;
- GC pause, clock drift, network partition и истечение TTL могут привести к двум владельцам lock;
- для критичных операций нужен fencing token или проверка version/constraint в source of truth.

### Fencing Tokens

Fencing token - монотонно растущий номер, который выдается при получении lock и проверяется ресурсом.

```sql
UPDATE accounts
SET balance = balance - 100,
    last_fencing_token = :token
WHERE id = :id
  AND last_fencing_token < :token;
```

Даже если старый процесс проснулся после pause и думает, что lock еще его, ресурс отклонит старый token.

## Idempotency

Идемпотентная операция может быть безопасно выполнена несколько раз с тем же эффектом, что и один раз.

Примеры:

- `PUT /users/42/email` с тем же email обычно idempotent;
- `POST /payments` без idempotency key обычно не idempotent;
- `POST /payments` с idempotency key может быть idempotent;
- `DELETE /resource/42` часто делают idempotent: повторный delete возвращает 204 или 404 в зависимости от API contract.

### Idempotency Key

Idempotency key - уникальный ключ операции, который клиент передает при повторяемом запросе. Сервер сохраняет результат первой обработки и возвращает его на повтор.

HTTP пример:

```http
POST /api/payments HTTP/1.1
Idempotency-Key: 01J2Z3K3Q4G5R6T7Y8U9I0O1P2
Content-Type: application/json

{"order_id": 123, "amount": 1000, "currency": "EUR"}
```

Таблица:

```sql
CREATE TABLE idempotency_keys (
    key text NOT NULL,
    user_id bigint NOT NULL,
    request_hash text NOT NULL,
    status text NOT NULL,
    response_code integer,
    response_body jsonb,
    locked_until timestamptz,
    created_at timestamptz NOT NULL DEFAULT now(),
    updated_at timestamptz NOT NULL DEFAULT now(),
    PRIMARY KEY (user_id, key)
);
```

Laravel пример:

```php
public function store(Request $request): JsonResponse
{
    $key = $request->header('Idempotency-Key');

    if ($key === null) {
        abort(400, 'Idempotency-Key is required');
    }

    $requestHash = hash('sha256', $request->getContent());

    return DB::transaction(function () use ($request, $key, $requestHash): JsonResponse {
        $record = IdempotencyKey::query()
            ->where('user_id', $request->user()->id)
            ->where('key', $key)
            ->lockForUpdate()
            ->first();

        if ($record !== null) {
            if ($record->request_hash !== $requestHash) {
                abort(409, 'Idempotency key reused with different request');
            }

            if ($record->status === 'completed') {
                return response()->json($record->response_body, $record->response_code);
            }

            abort(409, 'Request is already being processed');
        }

        IdempotencyKey::create([
            'user_id' => $request->user()->id,
            'key' => $key,
            'request_hash' => $requestHash,
            'status' => 'processing',
        ]);

        $payment = Payment::createFromRequest($request);

        IdempotencyKey::query()
            ->where('user_id', $request->user()->id)
            ->where('key', $key)
            ->update([
                'status' => 'completed',
                'response_code' => 201,
                'response_body' => ['payment_id' => $payment->id],
            ]);

        return response()->json(['payment_id' => $payment->id], 201);
    });
}
```

На практике создание платежа и сохранение ответа часто разделяют аккуратнее, особенно если есть внешний provider. Важно не держать DB transaction во время медленного HTTP-вызова. Тогда используют локальную запись операции, status machine и outbox.

Что хранить:

- key;
- scope: user, merchant, tenant или endpoint;
- hash тела запроса;
- status: processing/completed/failed;
- response code и body;
- created_at и TTL;
- ссылку на созданный resource.

Ловушки:

- key не привязан к user/tenant;
- один key используют с разным payload;
- сохраняют только факт обработки, но не результат;
- не очищают старые keys;
- key предсказуемый;
- нет unique constraint;
- idempotency key считают заменой business invariants.

## Exactly-Once Myth

В distributed systems exactly-once в end-to-end смысле почти всегда миф или маркетинговое упрощение.

Проблемы:

- сеть может доставить запрос, но клиент получит timeout;
- consumer может выполнить side effect и упасть до ack;
- producer может записать в БД, но упасть до publish;
- broker может переотправить message;
- внешний API может выполнить операцию, но не вернуть response.

Практичный ответ: проектируем под at-least-once delivery плюс idempotent handlers, deduplication и transactional boundaries.

## Delivery Semantics

### At-Least-Once

Message будет доставлен минимум один раз, но возможны дубли.

Плюсы:

- меньше риск потерять event;
- стандартная модель для RabbitMQ с manual ack;
- хорошо сочетается с retries.

Минусы:

- handlers должны быть idempotent;
- нужна deduplication;
- возможны повторные side effects.

RabbitMQ пример:

```php
$channel->basic_qos(null, 20, null);

$callback = function (AMQPMessage $message) use ($channel): void {
    try {
        handleMessage($message);
        $channel->basic_ack($message->getDeliveryTag());
    } catch (RetryableException $e) {
        $channel->basic_nack($message->getDeliveryTag(), false, true);
    } catch (Throwable $e) {
        $channel->basic_reject($message->getDeliveryTag(), false);
    }
};
```

Ack должен происходить после успешного commit локальной transaction.

### At-Most-Once

Message доставляется максимум один раз, но может потеряться.

Пример: auto-ack до обработки. Если consumer упал во время обработки, message уже считается доставленным.

Подходит для:

- metrics, где потеря допустима;
- fire-and-forget notifications низкой важности;
- ephemeral events.

Не подходит для:

- payments;
- order status changes;
- bonus accrual;
- critical integrations.

## Deduplication and Inbox

Deduplication - отсечение повторной обработки по stable identifier.

Inbox table:

```sql
CREATE TABLE inbox_messages (
    consumer_name text NOT NULL,
    message_id text NOT NULL,
    processed_at timestamptz NOT NULL DEFAULT now(),
    PRIMARY KEY (consumer_name, message_id)
);
```

Consumer:

```php
DB::transaction(function () use ($message): void {
    try {
        DB::table('inbox_messages')->insert([
            'consumer_name' => 'billing-service',
            'message_id' => $message->id,
            'processed_at' => now(),
        ]);
    } catch (UniqueConstraintViolationException $e) {
        return;
    }

    applyBusinessChange($message);
});
```

Ловушка: если вставить inbox record и затем выполнить side effect вне transaction, можно пометить message обработанным до реального эффекта. Для локальных DB-изменений inbox и business update должны быть в одной transaction.

Иногда отдельная inbox table не нужна, если business table уже имеет unique key:

```sql
CREATE UNIQUE INDEX bonus_accruals_event_id_unique
ON bonus_accruals (event_id);
```

## Outbox Pattern

Outbox решает dual write: надо одновременно изменить БД и опубликовать message. Нельзя надежно сделать это двумя независимыми операциями без протокола или паттерна.

Плохой вариант:

```php
$order->markPaid();
$order->save();
$rabbitMq->publish(new OrderPaid($order->id));
```

Если приложение упадет после `save()`, но до `publish()`, event потерян.

Outbox table:

```sql
CREATE TABLE outbox_messages (
    id bigserial PRIMARY KEY,
    aggregate_type text NOT NULL,
    aggregate_id text NOT NULL,
    event_type text NOT NULL,
    payload jsonb NOT NULL,
    headers jsonb NOT NULL DEFAULT '{}'::jsonb,
    available_at timestamptz NOT NULL DEFAULT now(),
    published_at timestamptz,
    attempts integer NOT NULL DEFAULT 0,
    created_at timestamptz NOT NULL DEFAULT now()
);

CREATE INDEX outbox_messages_unpublished_idx
ON outbox_messages (available_at, id)
WHERE published_at IS NULL;
```

Запись business change и event:

```php
DB::transaction(function () use ($order): void {
    $order->markPaid();
    $order->save();

    DB::table('outbox_messages')->insert([
        'aggregate_type' => 'order',
        'aggregate_id' => (string) $order->id,
        'event_type' => 'OrderPaid',
        'payload' => json_encode(['order_id' => $order->id], JSON_THROW_ON_ERROR),
        'headers' => json_encode(['message_id' => Str::uuid()->toString()], JSON_THROW_ON_ERROR),
    ]);
});
```

Publisher worker:

```php
DB::transaction(function () use ($publisher): void {
    $messages = DB::select(
        'SELECT *
         FROM outbox_messages
         WHERE published_at IS NULL
           AND available_at <= now()
         ORDER BY id
         FOR UPDATE SKIP LOCKED
         LIMIT 100'
    );

    foreach ($messages as $message) {
        $publisher->publish($message->event_type, $message->payload, $message->headers);

        DB::table('outbox_messages')
            ->where('id', $message->id)
            ->update(['published_at' => now()]);
    }
});
```

В реальных системах publish внутри DB transaction спорен: он держит lock во время сетевого вызова. Часто делают claiming записей, publish вне transaction и последующий mark as published. Это может дать повторную публикацию, поэтому downstream должен быть idempotent. Outbox обычно гарантирует at-least-once publish, не exactly-once.

### Outbox via CDC

Альтернатива polling worker - Change Data Capture, например Debezium читает PostgreSQL WAL и публикует outbox events в Kafka/RabbitMQ-compatible pipeline.

Плюсы:

- меньше ручного polling;
- хорошая throughput;
- ближе к transaction log.

Минусы:

- сложнее infrastructure;
- нужно версионировать event schemas;
- все равно нужны idempotent consumers.

## Inbox + Outbox Together

Комбинация:

- service A пишет business state и outbox event в одной transaction;
- publisher отправляет event в RabbitMQ;
- service B получает event;
- service B пишет inbox record и business state в одной transaction;
- service B ack'ает message после commit.

Так достигается практическая надежность при at-least-once delivery.

## Sagas

Saga - способ координации долгой business transaction между несколькими services без глобальной ACID transaction. Каждый шаг имеет локальную transaction и, при необходимости, compensating action.

Пример заказа:

1. Создать order в статусе `pending`.
2. Зарезервировать товар.
3. Авторизовать платеж.
4. Создать shipment.
5. Перевести order в `confirmed`.

Если shipment не создан, надо отменить авторизацию платежа и снять резерв.

### Choreography

Services реагируют на events друг друга без центрального координатора.

Плюсы:

- меньше центральной логики;
- weak coupling;
- естественно для event-driven systems.

Минусы:

- сложно понять полный flow;
- сложнее отлаживать;
- риск циклов events;
- business process размазан по services.

### Orchestration / Process Manager

Отдельный process manager хранит состояние процесса и явно отправляет commands.

Плюсы:

- flow виден в одном месте;
- проще контролировать retries, timeouts и compensations;
- удобно для сложных business processes.

Минусы:

- process manager становится важным component;
- больше явной infrastructure logic;
- надо проектировать состояние процесса.

Process manager table:

```sql
CREATE TABLE order_sagas (
    id uuid PRIMARY KEY,
    order_id bigint NOT NULL UNIQUE,
    state text NOT NULL,
    version integer NOT NULL DEFAULT 1,
    data jsonb NOT NULL DEFAULT '{}'::jsonb,
    deadline_at timestamptz,
    created_at timestamptz NOT NULL DEFAULT now(),
    updated_at timestamptz NOT NULL DEFAULT now()
);
```

Symfony Messenger idea:

```php
final class OrderSagaHandler
{
    public function __invoke(OrderPaid $event): void
    {
        $this->entityManager->wrapInTransaction(function () use ($event): void {
            $saga = $this->repository->getForUpdate($event->orderId);

            if (!$saga->canReserveStock()) {
                return;
            }

            $saga->markStockReservationRequested();
            $this->messageBus->dispatch(new ReserveStock($event->orderId));
        });
    }
}
```

Ловушка: dispatch внутри transaction может отправить message до commit, если transport не transactional. В Symfony и Laravel лучше связывать отправку внешних messages с outbox или after-commit механизмом, если он гарантирует нужную семантику.

Laravel queue after commit:

```php
DB::transaction(function () use ($order): void {
    $order->confirm();
    $order->save();

    SendOrderConfirmed::dispatch($order->id)->afterCommit();
});
```

Это полезно, но не полная замена outbox. Если process упадет после commit до фактической отправки job, гарантия зависит от queue driver и момента записи job.

## Consistency Models

### Strong Consistency

После успешной операции все читатели видят актуальное состояние. Обычно проще в рамках одной БД и одной transaction.

Подходит для:

- balances;
- limits;
- uniqueness;
- access rights;
- statuses, влияющих на деньги или юридические обязательства.

### Eventual Consistency

Система сходится к согласованному состоянию со временем.

Подходит для:

- read models;
- search index;
- analytics;
- notifications;
- caches;
- integrations, где допустима задержка.

Senior-ответ: eventual consistency не означает хаос. Нужны SLA задержки, monitoring lag, reconciliation job и понятные статусы для пользователя.

### Read Your Writes

Если пользователь сразу после записи читает из replica/read model, он может не увидеть изменение.

Решения:

- читать критичный результат из primary;
- sticky session/read-after-write window;
- optimistic UI/status `processing`;
- version или timestamp ожидания read model.

## Retries and Timeouts

Retries лечат временные ошибки, но могут усилить аварию.

### Timeout

У каждого внешнего вызова должен быть timeout. Бесконечное ожидание забивает workers и connection pool.

```php
$response = Http::timeout(3)
    ->connectTimeout(1)
    ->post($url, $payload);
```

### Retry with Backoff and Jitter

```php
retry(
    times: 3,
    callback: fn () => $client->send($request),
    sleepMilliseconds: fn (int $attempt) => random_int(50, 150) * $attempt,
);
```

Идея:

- не retry все подряд;
- отличать retryable и non-retryable errors;
- использовать exponential backoff;
- добавлять jitter;
- ограничивать общий deadline;
- сохранять idempotency key для повторяемых внешних операций.

### Retry Storm

Если зависимость деградирует, агрессивные retries могут добить ее. Нужны:

- circuit breaker;
- rate limit;
- bulkhead;
- backpressure;
- dead letter queue;
- graceful degradation.

## Backpressure

Backpressure не дает системе принимать работы больше, чем она может обработать.

Примеры:

- ограничить `prefetch` в RabbitMQ;
- ограничить количество queue workers;
- поставить rate limit на endpoint;
- возвращать `429 Too Many Requests` или `503 Retry-After`;
- ограничить DB connection pool;
- вводить bounded queue;
- откладывать jobs через delayed exchange или `available_at`;
- отключать неважные функции при перегрузке.

RabbitMQ prefetch:

```php
$channel->basic_qos(null, 10, null);
```

Если prefetch слишком большой, один consumer заберет много messages и будет долго держать их unacked. Если слишком маленький, снизится throughput.

## RabbitMQ Practical Notes

### Ack/Nack

- `ack` - message успешно обработан.
- `nack requeue=true` - вернуть в queue.
- `reject requeue=false` - отклонить, обычно в DLQ при настройке dead-letter exchange.
- Manual ack предпочтительнее для критичных задач.

### DLQ

Dead Letter Queue нужна для messages, которые не удалось обработать после ограниченного числа попыток.

Важно хранить:

- причину ошибки;
- количество попыток;
- исходный routing key;
- message id/correlation id;
- время первой и последней ошибки.

### Poison Message

Poison message - message, который всегда ломает consumer. Без DLQ он может бесконечно переобрабатываться и блокировать поток.

## Redis Practical Notes

Redis часто используют для cache, rate limit, lock и deduplication windows.

### SET NX PX

```text
SET key value NX PX 30000
```

Использовать для lock только с уникальным value и безопасным unlock.

### Deduplication Window

```php
$added = $redis->set('dedup:message:' . $messageId, '1', ['NX', 'EX' => 86400]);

if (!$added) {
    return;
}
```

Подходит для некритичных dedup windows. Для критичных операций лучше БД с durable unique constraint.

## PHP/Laravel/Symfony Pitfalls

### PHP-FPM

- Нет общей памяти между workers, если специально ее не использовать.
- Локальный static/cache внутри процесса не является distributed synchronization.
- File lock на одном сервере не работает на нескольких instances без общего FS с корректной семантикой.

### Laravel

- `firstOrCreate()` без unique constraint не защищает от race.
- `updateOrCreate()` тоже требует unique constraint.
- `withoutOverlapping()` для scheduler зависит от cache lock и выбранного store.
- Queue jobs могут быть выполнены повторно, handler должен быть idempotent.
- `ShouldBeUnique` снижает дубли jobs, но не заменяет business idempotency.
- Events и jobs лучше отправлять after commit или через outbox.

### Symfony

- Messenger handlers должны быть idempotent.
- Retry strategy не должна бесконечно обрабатывать poison messages.
- Doctrine UnitOfWork может отложить SQL до flush, поэтому важно понимать реальную границу transaction.
- Dispatch message из handler лучше связывать с outbox, если нужна надежная публикация.

## Typical Senior Answers

### Как защитить списание денег от гонок?

Я бы держал source of truth в PostgreSQL и защищал invariant на уровне БД. Для простого списания использовал бы atomic `UPDATE accounts SET balance = balance - :amount WHERE id = :id AND balance >= :amount` и проверку affected rows. Если операция сложнее и затрагивает несколько строк, использовал бы transaction и `SELECT ... FOR UPDATE` в стабильном порядке. Внешние вызовы не делал бы внутри transaction. Для повторных запросов добавил бы idempotency key и unique constraint.

### Почему нельзя просто поставить Redis lock вокруг платежа?

Redis lock может помочь снизить parallelism, но не должен быть единственной защитой correctness-critical invariant. TTL может истечь, process может зависнуть, сеть может разделиться. Для денег нужна защита в source of truth: constraints, atomic update, row lock, version или fencing token. Redis lock можно использовать как дополнительный efficiency lock.

### Как гарантировать, что event OrderPaid не потеряется?

Не делать dual write `save в БД + publish в broker` как две независимые операции. Записать изменение заказа и outbox message в одной DB transaction. Отдельный publisher читает outbox и публикует event. Так мы получаем at-least-once publish, поэтому consumers должны быть idempotent и иметь deduplication/inbox.

### Как обработать webhook от платежного провайдера?

Проверить подпись, использовать provider event id как deduplication key, записать inbox record и изменение платежа в одной transaction. Handler должен учитывать status model: переходы `pending -> paid` допустимы, повторный `paid` игнорируется, конфликтующие статусы разбираются по правилам provider. Ack/2xx отдавать только после успешного commit.

### Что делать, если consumer упал после side effect, но до ack?

Broker доставит message повторно. Поэтому side effect должен быть idempotent или защищенным unique constraint/inbox. Если side effect внешний, надо передавать idempotency key во внешний API или иметь reconciliation process.

### Как проектировать saga?

Сначала определить steps, local transactions, states, commands, events, timeouts и compensations. Для сложного процесса я бы выбрал process manager/orchestrator с state table и optimistic locking. Все исходящие commands писал бы через outbox. Каждый handler должен быть idempotent. Compensation не всегда rollback, часто это новая business operation: refund, cancel reservation, create adjustment.

## Interview Pitfalls

- Говорить `exactly once`, не объясняя границы гарантии.
- Полагаться на проверку `exists()` перед `insert()` без unique constraint.
- Держать DB transaction во время HTTP-запроса во внешний сервис.
- Считать Redis lock абсолютной гарантией корректности.
- Ack'ать RabbitMQ message до commit.
- Не иметь DLQ для poison messages.
- Делать бесконечные retries без backoff и jitter.
- Не различать command id, event id и idempotency key.
- Не хранить request hash для idempotency key.
- Не проектировать compensations в saga.
- Считать eventual consistency оправданием для отсутствия monitoring.
- Не учитывать read replica lag.
- Использовать local memory lock в horizontally scaled system.

## Mini-Practice

### Двойная оплата

Пользователь отправил `POST /payments`, получил timeout и повторил запрос.

Что описать:

- обязательный `Idempotency-Key`;
- unique constraint `(merchant_id, idempotency_key)`;
- request hash;
- status `processing/completed/failed`;
- локальная запись payment operation;
- внешний provider call с его idempotency key;
- reconciliation для unknown statuses;
- повтор возвращает результат первой операции.

### Начисление бонусов по событию

RabbitMQ доставляет `OrderCompleted` минимум один раз.

Что описать:

- message id/event id;
- inbox или unique index `bonus_accruals(event_id)`;
- начисление и dedup в одной transaction;
- ack после commit;
- DLQ после ограниченных retry;
- метрики дублей и ошибок.

### Резервирование остатков

Нужно не продать товара больше, чем есть на складе.

Что описать:

- atomic update `available = available - qty WHERE available >= qty`;
- или `SELECT ... FOR UPDATE` для сложной логики;
- unique constraint на reservation id/order id;
- expiration резерва;
- compensation при отмене заказа;
- load testing конфликтов.

### Интеграция нескольких сервисов

Order service, Inventory service, Payment service и Shipping service должны провести заказ.

Что описать:

- saga/process manager;
- состояния процесса;
- commands и events;
- outbox для исходящих messages;
- inbox для входящих;
- timeout для каждого шага;
- compensations: release stock, void/refund payment, cancel shipment;
- observability по correlation id.

## Design Checklist

- Определить invariants: что нельзя нарушить ни при каких условиях.
- Выбрать source of truth.
- Перенести critical checks в БД через constraints или atomic update.
- Определить transaction boundaries.
- Не держать transactions во время медленных external calls.
- Выбрать locking strategy: optimistic, pessimistic, advisory или distributed lock.
- Для повторяемых HTTP commands добавить idempotency key.
- Для messages принять at-least-once как базовую модель.
- Сделать handlers idempotent.
- Добавить inbox/deduplication для входящих messages.
- Добавить outbox для надежной публикации events.
- Спроектировать retry policy с backoff, jitter и max attempts.
- Настроить DLQ для poison messages.
- Добавить timeout и deadline для external calls.
- Продумать backpressure и rate limits.
- Для saga описать states, timeouts и compensations.
- Добавить correlation id, structured logs, metrics lag/retry/error/dedup.
- Добавить reconciliation jobs для external systems.
- Проверить поведение при падении между каждым шагом.

## Self-Check

- Могу ли я объяснить lost update на SQL/PHP примере?
- Понимаю ли разницу между optimistic и pessimistic locking?
- Могу ли объяснить, почему `firstOrCreate()` без unique index небезопасен?
- Знаю ли, когда использовать `FOR UPDATE`, `NOWAIT`, `SKIP LOCKED`?
- Могу ли объяснить, почему Redis lock не заменяет constraint в БД?
- Могу ли описать Redlock caveats без религиозного спора?
- Могу ли спроектировать idempotency key storage?
- Могу ли объяснить exactly-once myth?
- Понимаю ли разницу между at-least-once и at-most-once?
- Могу ли написать schema outbox table?
- Понимаю ли, почему outbox обычно дает at-least-once publish?
- Могу ли объяснить inbox pattern?
- Могу ли спроектировать saga с compensations?
- Умею ли говорить про retries, timeouts, jitter и DLQ?
- Могу ли объяснить backpressure на примере RabbitMQ prefetch?

## Дополнительное чтение

- Martin Fowler, Patterns of Distributed Systems: <https://martinfowler.com/articles/patterns-of-distributed-systems/>
- Martin Fowler, Saga: <https://martinfowler.com/eaaDev/Saga.html>
- Microservices.io, Transactional Outbox: <https://microservices.io/patterns/data/transactional-outbox.html>
- Designing Data-Intensive Applications, Martin Kleppmann: <https://dataintensive.net/>
- Martin Kleppmann, How to do distributed locking: <https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html>
- Redis, Distributed locks with Redis: <https://redis.io/docs/latest/develop/clients/patterns/distributed-locks/>
- PostgreSQL, Explicit Locking: <https://www.postgresql.org/docs/current/explicit-locking.html>
- PostgreSQL, Transaction Isolation: <https://www.postgresql.org/docs/current/transaction-iso.html>
- PostgreSQL, Advisory Locks: <https://www.postgresql.org/docs/current/explicit-locking.html#ADVISORY-LOCKS>
- RabbitMQ, Consumer acknowledgements and publisher confirms: <https://www.rabbitmq.com/docs/confirms>
- RabbitMQ, Consumer prefetch: <https://www.rabbitmq.com/docs/consumer-prefetch>
- RabbitMQ, Dead Letter Exchanges: <https://www.rabbitmq.com/docs/dlx>
- Laravel, Atomic Locks: <https://laravel.com/docs/cache#atomic-locks>
- Laravel, Queues and database transactions: <https://laravel.com/docs/queues#jobs-and-database-transactions>
- Laravel, Scheduler without overlapping: <https://laravel.com/docs/scheduling#preventing-task-overlaps>
- Symfony Lock Component: <https://symfony.com/doc/current/components/lock.html>
- Symfony Messenger Component: <https://symfony.com/doc/current/messenger.html>
- Debezium Outbox Event Router: <https://debezium.io/documentation/reference/stable/transformations/outbox-event-router.html>
