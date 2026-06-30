# Concurrency and distributed systems: locks, idempotency, outbox, sagas

Гайд для подготовки к интервью на позицию Lead/Senior PHP backend developer. Фокус: как рассуждать о гонках, блокировках, идемпотентности, доставке сообщений, outbox/inbox и сагах в реальных системах на PHP, Laravel, Symfony, PostgreSQL, Redis и RabbitMQ.

## Что проверяют на интервью

Интервьюер обычно хочет понять не знание названий паттернов, а способность проектировать систему, которая корректно работает при параллельных запросах, сбоях сети, ретраях, падениях воркеров, дублях сообщений и частичной недоступности зависимостей.

Хороший Senior-ответ почти всегда содержит:

- где находится источник истины;
- какие инварианты нельзя нарушить;
- какая модель консистентности допустима;
- какие операции идемпотентны, а какие нет;
- где нужны транзакции и блокировки;
- что произойдет при таймауте, повторе, падении процесса и дубле сообщения;
- как система восстанавливается;
- какие метрики и алерты нужны.

## Базовые понятия

### Concurrency и parallelism

Concurrency - это про несколько задач, которые логически выполняются в перекрывающиеся периоды времени. Parallelism - физическое выполнение одновременно, например на разных CPU core или разных воркерах.

В PHP backend concurrency обычно появляется не из потоков внутри одного процесса, а из:

- нескольких PHP-FPM воркеров;
- нескольких queue workers;
- нескольких consumer'ов RabbitMQ;
- нескольких cron/job процессов;
- горизонтального масштабирования приложения;
- повторных HTTP-запросов клиента;
- ретраев от балансировщика, клиента, брокера или scheduler'а.

### Race condition

Race condition - ошибка, при которой результат зависит от непредсказуемого порядка выполнения параллельных операций.

Классический пример: списание остатка.

```php
$account = Account::find($id);

if ($account->balance >= 100) {
    $account->balance -= 100;
    $account->save();
}
```

Если два запроса одновременно прочитали balance = 100, оба могут пройти проверку и оба сохранить списание. Инвариант `balance >= 0` нарушен.

### Critical section

Critical section - участок кода, где выполняется чтение/проверка/изменение общего состояния и где нельзя допустить неконтролируемого параллельного выполнения.

Важно: критическая секция обычно должна быть как можно меньше. Нельзя держать блокировку во время HTTP-вызова внешнего сервиса, отправки email или долгой CPU-операции, если это можно вынести за пределы транзакции.

## Типовые гонки в backend

### Lost update

Два процесса читают одно состояние, оба изменяют, последний `save()` затирает результат первого.

Решения:

- атомарный SQL `UPDATE ... SET value = value + 1`;
- optimistic locking через `version`;
- pessimistic lock `SELECT ... FOR UPDATE`;
- constraint и повтор транзакции.

### Check-then-act

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

### Double submit

Пользователь дважды нажал кнопку оплаты, мобильное приложение повторило запрос после таймаута, gateway отправил webhook два раза.

Решения:

- idempotency key;
- unique constraint на бизнес-ключ;
- статусная модель;
- inbox/deduplication для входящих событий.

### Duplicate message processing

Брокер доставил одно сообщение несколько раз, consumer упал после обработки, но до ack.

Решения:

- обработчик должен быть идемпотентным;
- inbox table с `message_id`;
- уникальные ограничения;
- ack только после commit.

## Оптимистическая блокировка

Optimistic locking предполагает, что конфликты редки. Мы не блокируем строку при чтении, но при записи проверяем, что версия не изменилась.

### PostgreSQL пример

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

### Когда подходит

- конфликты редки;
- пользователь может повторить действие;
- важна высокая пропускная способность;
- можно показать ошибку `record was changed, reload and retry`;
- можно безопасно повторить команду.

### Ловушки

- забыли включить `version` в `WHERE`;
- обновляют через ORM `save()` без проверки версии;
- делают несколько связанных изменений, но версионируют только одну сущность;
- retry без ограничения превращается в шторм;
- конфликт скрывают и перезаписывают чужие изменения.

## Пессимистическая блокировка

Pessimistic locking предполагает, что конфликт вероятен или цена ошибки высока. Мы явно блокируем строки до конца транзакции.

### Laravel пример

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

В PostgreSQL это соответствует `SELECT ... FOR UPDATE`.

### Symfony/Doctrine пример

```php
$entityManager->wrapInTransaction(function () use ($entityManager, $accountId, $amount): void {
    $account = $entityManager->find(
        Account::class,
        $accountId,
        LockMode::PESSIMISTIC_WRITE
    );

    if ($account->balance() < $amount) {
        throw new InsufficientFundsException();
    }

    $account->withdraw($amount);
});
```

### Когда подходит

- деньги, остатки, лимиты, квоты;
- нельзя допустить нарушение инварианта;
- критическая секция короткая;
- конфликт ожидаем;
- проще подождать блокировку, чем обрабатывать конфликт версии.

### Ловушки

- блокировка работает только внутри транзакции;
- ORM lazy loading может неявно расширить транзакцию;
- долгие операции внутри транзакции приводят к lock contention;
- разный порядок блокировок приводит к deadlock;
- `SELECT` без `FOR UPDATE` не защищает от параллельного изменения;
- блокировка строки не защищает от вставки новой строки, если инвариант относится к диапазону.

## DB locks в PostgreSQL

### Row-level locks

Основные варианты:

- `FOR UPDATE` - блокирует строки для изменения;
- `FOR NO KEY UPDATE` - слабее, если не меняется key;
- `FOR SHARE` - shared lock для чтения;
- `FOR KEY SHARE` - защита внешних ключей.

Пример:

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

### NOWAIT и SKIP LOCKED

`NOWAIT` сразу падает, если строка уже заблокирована.

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

### Advisory locks

PostgreSQL advisory locks - пользовательские блокировки по числовому ключу.

```sql
SELECT pg_try_advisory_lock(12345);
SELECT pg_advisory_unlock(12345);
```

Транзакционный вариант безопаснее для большинства backend-сценариев:

```sql
SELECT pg_try_advisory_xact_lock(12345);
```

Когда использовать:

- защита cron/job от параллельного запуска;
- блокировка по бизнес-ключу, когда строки еще нет;
- coarse-grained lock для редких операций.

Ловушки:

- надо стабильно хешировать бизнес-ключ;
- session-level lock можно забыть освободить;
- advisory lock не заменяет constraints;
- не видно напрямую на уровне доменной модели.

### Isolation levels

PostgreSQL по умолчанию использует `READ COMMITTED`. Это нормально для большинства операций, если инварианты защищены constraints, row locks или атомарными update.

`REPEATABLE READ` дает стабильный снимок, но не решает все бизнес-инварианты автоматически.

`SERIALIZABLE` может предотвращать более широкий класс аномалий, но требует обработки serialization failure и retry транзакции.

Senior-ответ: isolation level - не магия. Надо понимать конкретный инвариант, тип аномалии и цену retry.

## Атомарные операции и constraints

Самая надежная блокировка часто вообще не выглядит как lock в коде.

### Атомарный UPDATE

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
- инвариант проверяется в БД.

### Unique constraint как защита от гонок

```sql
CREATE UNIQUE INDEX payments_idempotency_key_unique
ON payments (merchant_id, idempotency_key);
```

Это лучше, чем `exists()` перед `insert()`.

### Partial unique index

Например, только один активный subscription на пользователя.

```sql
CREATE UNIQUE INDEX subscriptions_one_active_per_user
ON subscriptions (user_id)
WHERE status = 'active';
```

## Distributed locks

Distributed lock нужен, когда несколько процессов на разных машинах должны эксклюзивно выполнить действие, а обычной DB-транзакции недостаточно или ресурс не находится в одной БД.

Примеры:

- один scheduler job на весь кластер;
- один reindex на tenant;
- защита внешнего API от параллельных несовместимых вызовов;
- leader election для легковесного процесса.

### Redis lock: минимальный корректный вариант

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
- unlock должен быть атомарным Lua-скриптом;
- TTL должен быть больше ожидаемой критической секции с запасом.

### Laravel atomic locks

```php
Cache::lock('tenant:42:reindex', 30)->block(5, function (): void {
    reindexTenant(42);
});
```

Важно проверить используемый cache driver. Для распределенной блокировки нужны Redis, Memcached, DynamoDB или database driver с понятной семантикой. `array` и локальный file cache не подходят для нескольких инстансов.

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

Symfony Lock поддерживает разные stores: Redis, PDO, PostgreSQL advisory locks, Semaphore и другие. На интервью важно объяснить, какой store выбран и почему.

### Redlock caveats

Redlock - алгоритм distributed lock поверх нескольких независимых Redis-инстансов. Вокруг него есть известная дискуссия: antirez предложил алгоритм, Martin Kleppmann критиковал его для correctness-critical систем.

Главная мысль для Senior-ответа:

- Redis lock можно использовать для efficiency locks: не запускать лишнюю работу, уменьшить дубли, защитить дорогую операцию;
- нельзя безоговорочно полагаться на Redis lock для correctness-critical инвариантов вроде денег, если нет дополнительной защиты;
- GC pause, clock drift, network partition и истечение TTL могут привести к тому, что два процесса считают себя владельцами lock;
- для критичных операций нужен fencing token или проверка версии/constraint в источнике истины.

### Fencing tokens

Fencing token - монотонно растущий номер, который выдается при получении lock и проверяется ресурсом.

Пример идеи:

```sql
UPDATE accounts
SET balance = balance - 100,
    last_fencing_token = :token
WHERE id = :id
  AND last_fencing_token < :token;
```

Даже если старый процесс проснулся после паузы и думает, что lock еще его, ресурс отклонит старый token.

## Идемпотентность

Идемпотентная операция может быть безопасно выполнена несколько раз с тем же эффектом, что и один раз.

Примеры:

- `PUT /users/42/email` с одним и тем же email обычно идемпотентен;
- `POST /payments` без idempotency key обычно не идемпотентен;
- `POST /payments` с idempotency key может быть идемпотентным;
- `DELETE /resource/42` часто делают идемпотентным: повторный delete возвращает 204 или 404 в зависимости от API-контракта.

### Idempotency key

Idempotency key - уникальный ключ операции, который клиент передает при повторяемом запросе. Сервер сохраняет результат первой обработки и возвращает его на повтор.

HTTP пример:

```http
POST /api/payments HTTP/1.1
Idempotency-Key: 01J2Z3K3Q4G5R6T7Y8U9I0O1P2
Content-Type: application/json

{"order_id": 123, "amount": 1000, "currency": "EUR"}
```

### Таблица idempotency keys

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

### Laravel пример обработки

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

На практике создание платежа и сохранение ответа часто разделяют аккуратнее, особенно если есть внешний payment provider. Важно не держать DB-транзакцию во время медленного внешнего HTTP-вызова. Тогда используют локальную запись операции, статусную машину и outbox.

### Что хранить

- key;
- scope: user, merchant, tenant или endpoint;
- hash тела запроса;
- статус: processing/completed/failed;
- код и тело ответа;
- created_at и TTL для очистки;
- ссылку на созданный ресурс.

### Ловушки

- key не привязан к пользователю или tenant;
- один key используют с разным payload;
- сохраняют только факт обработки, но не результат;
- не очищают старые keys;
- делают key предсказуемым;
- не защищают таблицу unique constraint;
- считают, что idempotency key заменяет бизнес-инварианты.

## Exactly-once myth

В распределенных системах exactly-once в end-to-end смысле почти всегда миф или маркетинговое упрощение.

Можно получить exactly-once processing внутри ограниченной системы при строгих условиях, но между HTTP-клиентом, сервисом, БД, брокером, consumer'ом и внешним API всегда остаются окна неопределенности.

Практичный Senior-ответ:

- сеть может доставить запрос, но клиент получит timeout;
- consumer может выполнить side effect и упасть до ack;
- producer может записать в БД, но упасть до publish;
- broker может переотправить сообщение;
- внешний API может выполнить операцию, но не вернуть ответ.

Поэтому проектируем под at-least-once delivery плюс идемпотентные handlers, deduplication и transactional boundaries.

## At-least-once и at-most-once delivery

### At-least-once

Сообщение будет доставлено минимум один раз, но возможны дубли.

Плюсы:

- меньше риск потерять событие;
- стандартная модель для RabbitMQ с manual ack;
- хорошо сочетается с retries.

Минусы:

- handlers должны быть идемпотентными;
- нужна дедупликация;
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

Ack должен происходить после успешного commit локальной транзакции.

### At-most-once

Сообщение доставляется максимум один раз, но может потеряться.

Пример: auto-ack до обработки. Если consumer упал во время обработки, сообщение уже считается доставленным.

Подходит для:

- метрик, где потеря допустима;
- fire-and-forget уведомлений низкой важности;
- ephemeral событий.

Не подходит для:

- платежей;
- изменения статусов заказов;
- начисления бонусов;
- критичных интеграций.

## Deduplication

Deduplication - отсечение повторной обработки по стабильному идентификатору.

### Inbox table

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

Ловушка: если вставить inbox record и затем выполнить side effect вне транзакции, можно пометить сообщение обработанным до реального эффекта. Для локальных DB-изменений inbox и business update должны быть в одной транзакции.

### Дедупликация через business key

Иногда отдельная inbox table не нужна, если бизнес-таблица уже имеет уникальный ключ.

```sql
CREATE UNIQUE INDEX bonus_accruals_event_id_unique
ON bonus_accruals (event_id);
```

Тогда повторное событие не создаст второе начисление.

## Outbox pattern

Outbox решает проблему dual write: надо одновременно изменить БД и опубликовать сообщение. Нельзя сделать это надежно двумя независимыми операциями без протокола или паттерна.

Плохой вариант:

```php
$order->markPaid();
$order->save();

$rabbitMq->publish(new OrderPaid($order->id));
```

Если приложение упадет после `save()`, но до `publish()`, событие потеряно.

### Outbox table

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

### Запись бизнес-изменения и события

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

### Publisher worker

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

В реальных системах publish внутри DB-транзакции спорен: он держит lock во время сетевого вызова. Часто делают claiming записей, publish вне транзакции и последующий mark as published. Это может дать повторную публикацию, поэтому downstream должен быть идемпотентным. Outbox обычно гарантирует at-least-once publish, не exactly-once.

### Outbox через CDC

Альтернатива polling worker - Change Data Capture, например Debezium читает PostgreSQL WAL и публикует outbox events в Kafka/RabbitMQ-compatible pipeline.

Плюсы:

- меньше ручного polling;
- хорошая пропускная способность;
- ближе к транзакционному логу.

Минусы:

- сложнее инфраструктура;
- нужно версионировать схемы событий;
- все равно нужны идемпотентные consumers.

## Inbox pattern

Inbox защищает consumer от повторной обработки входящих сообщений. Часто используется вместе с outbox.

Комбинация:

- service A пишет business state и outbox event в одной транзакции;
- publisher отправляет событие в RabbitMQ;
- service B получает событие;
- service B пишет inbox record и business state в одной транзакции;
- service B ack'ает сообщение после commit.

Так достигается практическая надежность при at-least-once delivery.

## Sagas

Saga - способ координации долгой бизнес-транзакции между несколькими сервисами без глобальной ACID-транзакции. Каждый шаг имеет локальную транзакцию и, при необходимости, compensating action.

Пример заказа:

1. Создать order в статусе `pending`.
2. Зарезервировать товар.
3. Авторизовать платеж.
4. Создать shipment.
5. Перевести order в `confirmed`.

Если shipment не создан, надо отменить авторизацию платежа и снять резерв.

### Choreography

Сервисы реагируют на события друг друга без центрального координатора.

Плюсы:

- меньше центральной логики;
- слабая связанность;
- естественно для event-driven систем.

Минусы:

- сложно понять полный flow;
- сложнее отлаживать;
- риск циклов событий;
- бизнес-процесс размазан по сервисам.

### Orchestration / process manager

Отдельный process manager хранит состояние процесса и явно отправляет команды.

Плюсы:

- flow виден в одном месте;
- проще контролировать retries, timeouts и компенсации;
- удобно для сложных бизнес-процессов.

Минусы:

- process manager становится важным компонентом;
- больше явной инфраструктурной логики;
- надо проектировать состояние процесса.

### Process manager таблица

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

### Symfony Messenger пример идеи

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

Важная ловушка: dispatch внутри транзакции может отправить сообщение до commit, если transport не транзакционный. В Symfony и Laravel лучше связывать отправку внешних сообщений с outbox или after-commit механизмом, если он гарантирует нужную семантику.

### Laravel queue after commit

```php
DB::transaction(function () use ($order): void {
    $order->confirm();
    $order->save();

    SendOrderConfirmed::dispatch($order->id)->afterCommit();
});
```

Это полезно, но не полная замена outbox. Если процесс упадет после commit до фактической отправки job, гарантия зависит от реализации queue driver и момента записи job.

## Consistency

### Strong consistency

После успешной операции все читатели видят актуальное состояние. Обычно проще в рамках одной БД и одной транзакции.

Подходит для:

- балансов;
- лимитов;
- уникальности;
- прав доступа;
- статусов, влияющих на деньги или юридические обязательства.

### Eventual consistency

Система сходится к согласованному состоянию со временем.

Подходит для:

- read models;
- поискового индекса;
- аналитики;
- уведомлений;
- кешей;
- интеграций, где допустима задержка.

Senior-ответ: eventual consistency не означает хаос. Нужны SLA задержки, мониторинг lag, reconciliation job и понятные статусы для пользователя.

### Read your writes

Если пользователь сразу после записи читает из replica/read model, он может не увидеть свое изменение. Решения:

- читать критичный результат из primary;
- sticky session/read-after-write window;
- показывать optimistic UI/status `processing`;
- использовать version или timestamp ожидания read model.

## Retries и timeouts

Retries лечат временные ошибки, но могут усилить аварию.

### Timeout

У каждого внешнего вызова должен быть timeout. Бесконечное ожидание забивает воркеры и connection pool.

```php
$response = Http::timeout(3)
    ->connectTimeout(1)
    ->post($url, $payload);
```

### Retry с backoff и jitter

```php
retry(
    times: 3,
    callback: fn () => $client->send($request),
    sleepMilliseconds: fn (int $attempt) => random_int(50, 150) * $attempt
);
```

Идея:

- не ретраить все подряд;
- отличать retryable и non-retryable ошибки;
- использовать exponential backoff;
- добавлять jitter;
- ограничивать общий deadline;
- сохранять idempotency key для повторяемых внешних операций.

### Retry storm

Если зависимость деградирует, агрессивные retries могут добить ее. Нужны:

- circuit breaker;
- rate limit;
- bulkhead;
- backpressure;
- dead letter queue;
- graceful degradation.

## Backpressure

Backpressure - механизм, который не дает системе принимать работы больше, чем она может обработать.

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

Если prefetch слишком большой, один consumer заберет много сообщений и будет долго держать их unacked. Если слишком маленький, снизится throughput.

## RabbitMQ практические моменты

### Ack/Nack

- `ack` - сообщение успешно обработано;
- `nack requeue=true` - вернуть в очередь;
- `reject requeue=false` - отклонить, обычно в DLQ при настройке dead-letter exchange;
- manual ack предпочтительнее для критичных задач.

### DLQ

Dead Letter Queue нужна для сообщений, которые не удалось обработать после ограниченного числа попыток.

Важно хранить:

- причину ошибки;
- количество попыток;
- исходный routing key;
- message id/correlation id;
- время первой и последней ошибки.

### Poison message

Poison message - сообщение, которое всегда ломает consumer. Без DLQ оно может бесконечно переобрабатываться и блокировать поток.

## Redis практические моменты

Redis часто используют для cache, rate limit, lock и deduplication windows.

### SET NX PX

```text
SET key value NX PX 30000
```

Использовать для lock только с уникальным value и безопасным unlock.

### Deduplication window

```php
$added = $redis->set('dedup:message:' . $messageId, '1', ['NX', 'EX' => 86400]);

if (!$added) {
    return;
}
```

Подходит для некритичных dedup window. Для критичных операций лучше БД с durable unique constraint.

## PHP/Laravel/Symfony pitfalls

### PHP-FPM

- Нет общей памяти между worker'ами, если специально ее не использовать.
- Локальный static/cache внутри процесса не является распределенной синхронизацией.
- File lock на одном сервере не работает на нескольких инстансах без общего FS с корректной семантикой.

### Laravel

- `firstOrCreate()` без unique constraint не защищает от гонки.
- `updateOrCreate()` тоже требует unique constraint.
- `withoutOverlapping()` для scheduler зависит от cache lock и выбранного store.
- Queue jobs могут быть выполнены повторно, handler должен быть идемпотентным.
- `ShouldBeUnique` снижает дубли jobs, но не заменяет бизнес-idempotency.
- События и jobs лучше отправлять after commit или через outbox.

### Symfony

- Messenger handlers должны быть идемпотентными.
- Retry strategy не должна бесконечно молотить poison messages.
- Doctrine UnitOfWork может отложить SQL до flush, поэтому важно понимать реальную границу транзакции.
- Dispatch message из handler'а лучше связывать с outbox, если нужна надежная публикация.

## Типичные senior answers

### Как защитить списание денег от гонок?

Я бы держал источник истины в PostgreSQL и защищал инвариант на уровне БД. Для простого списания использовал бы атомарный `UPDATE accounts SET balance = balance - :amount WHERE id = :id AND balance >= :amount` и проверку affected rows. Если операция сложнее и затрагивает несколько строк, использовал бы транзакцию и `SELECT ... FOR UPDATE` в стабильном порядке. Внешние вызовы не делал бы внутри транзакции. Для повторных запросов добавил бы idempotency key и unique constraint.

### Почему нельзя просто поставить Redis lock вокруг платежа?

Redis lock может помочь снизить параллелизм, но не должен быть единственной защитой correctness-critical инварианта. TTL может истечь, процесс может зависнуть, сеть может разделиться. Для денег нужна защита в источнике истины: constraints, atomic update, row lock, version или fencing token. Redis lock можно использовать как дополнительный efficiency lock.

### Как гарантировать, что событие OrderPaid не потеряется?

Не делать dual write `save в БД + publish в брокер` как две независимые операции. Записать изменение заказа и outbox message в одной DB-транзакции. Отдельный publisher читает outbox и публикует событие. Так мы получаем at-least-once publish, поэтому consumers должны быть идемпотентными и иметь deduplication/inbox.

### Как обработать webhook от платежного провайдера?

Проверить подпись, использовать provider event id как deduplication key, записать inbox record и изменение платежа в одной транзакции. Обработчик должен учитывать статусную модель: переходы `pending -> paid` допустимы, повторный `paid` игнорируется, конфликтующие статусы разбираются по правилам провайдера. Ack/2xx отдавать только после успешного commit.

### Что делать, если consumer упал после выполнения side effect, но до ack?

Брокер доставит сообщение повторно. Поэтому side effect должен быть идемпотентным или защищенным unique constraint/inbox. Если side effect внешний, надо передавать idempotency key во внешний API или иметь reconciliation process.

### Как проектировать сагу?

Сначала определить шаги, локальные транзакции, состояния, команды, события, timeout'ы и компенсации. Для сложного процесса я бы выбрал process manager/orchestrator с таблицей состояния и optimistic locking. Все исходящие команды писал бы через outbox. Каждый handler должен быть идемпотентным. Компенсация не всегда rollback, часто это новая бизнес-операция: refund, cancel reservation, create adjustment.

## Частые ловушки на интервью

- Говорить `exactly once`, не объясняя границы гарантии.
- Полагаться на проверку `exists()` перед `insert()` без unique constraint.
- Держать DB-транзакцию во время HTTP-запроса во внешний сервис.
- Считать Redis lock абсолютной гарантией корректности.
- Ack'ать RabbitMQ message до commit.
- Не иметь DLQ для poison messages.
- Делать бесконечные retries без backoff и jitter.
- Не различать command id, event id и idempotency key.
- Не хранить request hash для idempotency key.
- Не проектировать компенсации в saga.
- Считать eventual consistency оправданием для отсутствия мониторинга.
- Не учитывать read replica lag.
- Использовать локальный memory lock в горизонтально масштабированной системе.

## Мини-практика

### Задача 1: двойная оплата

Пользователь отправил `POST /payments`, получил timeout и повторил запрос. Первый запрос мог успешно дойти до сервера.

Что описать в решении:

- обязательный `Idempotency-Key`;
- unique constraint `(merchant_id, idempotency_key)`;
- request hash;
- статус `processing/completed/failed`;
- локальная запись payment operation;
- внешний provider call с его idempotency key;
- reconciliation для неизвестных статусов;
- повтор возвращает результат первой операции.

### Задача 2: начисление бонусов по событию

RabbitMQ доставляет `OrderCompleted` минимум один раз.

Что описать:

- message id/event id;
- inbox или unique index `bonus_accruals(event_id)`;
- начисление и dedup в одной транзакции;
- ack после commit;
- DLQ после ограниченных retry;
- метрики дублей и ошибок.

### Задача 3: резервирование остатков

Нужно не продать товара больше, чем есть на складе.

Что описать:

- атомарный update `available = available - qty WHERE available >= qty`;
- или `SELECT ... FOR UPDATE` для сложной логики;
- unique constraint на reservation id/order id;
- expiration резерва;
- компенсация при отмене заказа;
- нагрузочное тестирование конфликтов.

### Задача 4: интеграция нескольких сервисов

Order service, Inventory service, Payment service и Shipping service должны провести заказ.

Что описать:

- saga/process manager;
- состояния процесса;
- команды и события;
- outbox для исходящих сообщений;
- inbox для входящих;
- timeout для каждого шага;
- компенсации: release stock, void/refund payment, cancel shipment;
- observability по correlation id.

## Self-check

- Могу ли я объяснить lost update на простом SQL/PHP примере?
- Понимаю ли я разницу между optimistic и pessimistic locking?
- Могу ли я объяснить, почему `firstOrCreate()` без unique index небезопасен?
- Знаю ли я, когда использовать `FOR UPDATE`, `NOWAIT`, `SKIP LOCKED`?
- Могу ли я объяснить, почему Redis lock не заменяет constraint в БД?
- Могу ли я описать Redlock caveats без ухода в религиозный спор?
- Могу ли я спроектировать idempotency key storage?
- Могу ли я объяснить exactly-once myth?
- Понимаю ли я разницу между at-least-once и at-most-once?
- Могу ли я написать схему outbox table?
- Понимаю ли я, почему outbox обычно дает at-least-once publish?
- Могу ли я объяснить inbox pattern?
- Могу ли я спроектировать saga с компенсациями?
- Умею ли я говорить про retries, timeouts, jitter и DLQ?
- Могу ли я объяснить backpressure на примере RabbitMQ prefetch?

## Checklist для проектирования

- Определить инварианты: что нельзя нарушить ни при каких условиях.
- Выбрать источник истины.
- Перенести критичные проверки в БД через constraints или atomic update.
- Определить границы транзакций.
- Не держать транзакции во время медленных внешних вызовов.
- Выбрать locking strategy: optimistic, pessimistic, advisory или distributed lock.
- Для повторяемых HTTP-команд добавить idempotency key.
- Для сообщений принять at-least-once как базовую модель.
- Сделать handlers идемпотентными.
- Добавить inbox/deduplication для входящих сообщений.
- Добавить outbox для надежной публикации событий.
- Спроектировать retry policy с backoff, jitter и max attempts.
- Настроить DLQ для poison messages.
- Добавить timeout и deadline для внешних вызовов.
- Продумать backpressure и rate limits.
- Для saga описать состояния, timeout'ы и компенсации.
- Добавить correlation id, structured logs, метрики lag/retry/error/dedup.
- Добавить reconciliation jobs для внешних систем.
- Проверить поведение при падении между каждым шагом.

## Ссылки

- Martin Fowler: Patterns of Distributed Systems - https://martinfowler.com/articles/patterns-of-distributed-systems/
- Martin Fowler: Saga - https://martinfowler.com/eaaDev/Saga.html
- Martin Fowler: Transactional Outbox - https://microservices.io/patterns/data/transactional-outbox.html
- Designing Data-Intensive Applications, Martin Kleppmann - https://dataintensive.net/
- Martin Kleppmann: How to do distributed locking - https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html
- Redis: Distributed locks with Redis - https://redis.io/docs/latest/develop/clients/patterns/distributed-locks/
- PostgreSQL: Explicit Locking - https://www.postgresql.org/docs/current/explicit-locking.html
- PostgreSQL: Transaction Isolation - https://www.postgresql.org/docs/current/transaction-iso.html
- PostgreSQL: Advisory Locks - https://www.postgresql.org/docs/current/explicit-locking.html#ADVISORY-LOCKS
- RabbitMQ: Consumer acknowledgements and publisher confirms - https://www.rabbitmq.com/docs/confirms
- RabbitMQ: Consumer prefetch - https://www.rabbitmq.com/docs/consumer-prefetch
- RabbitMQ: Dead Letter Exchanges - https://www.rabbitmq.com/docs/dlx
- Laravel: Atomic Locks - https://laravel.com/docs/cache#atomic-locks
- Laravel: Queues and database transactions - https://laravel.com/docs/queues#jobs-and-database-transactions
- Symfony: Lock Component - https://symfony.com/doc/current/components/lock.html
- Symfony: Messenger Component - https://symfony.com/doc/current/messenger.html
