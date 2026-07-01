# Redis advanced: caching, locks, streams, persistence, cluster

Гайд для подготовки к собеседованию Lead/Senior PHP backend developer. Фокус: не только знать команды Redis, но и уметь объяснить trade-off, отказоустойчивость, консистентность, диагностику и интеграцию с PHP-приложением.

## Что должен знать Senior

Redis - это in-memory data store с богатым набором структур данных, атомарными операциями, Lua-скриптами, механизмами репликации, persistence, Sentinel и Cluster. На собеседовании важно не продавать Redis как универсальную БД, а показывать понимание границ:

- Redis очень быстрый, потому что данные в памяти, операции в основном single-threaded на event loop, сетевой протокол простой.
- Redis отлично подходит для кеша, rate limiting, счетчиков, очередей легких задач, distributed coordination с оговорками, pub/sub, streams.
- Redis хуже подходит для данных, где нельзя потерять ни одной записи без дополнительного durability-дизайна.
- Redis Cluster дает horizontal scaling, но усложняет multi-key операции, Lua и транзакции.
- Distributed locks в Redis - компромисс, а не замена ZooKeeper/etcd/Consul для критической координации.

## Базовая модель Redis

Redis хранит ключи в памяти. Ключ - binary-safe строка. Значение может быть одной из структур данных. У каждого ключа может быть TTL. Удаление expired keys происходит лениво при доступе и активно фоновым циклом.

Важные свойства:

- Большинство команд атомарны относительно одного Redis-инстанса.
- Redis обрабатывает команды последовательно в основном потоке, но I/O и фоновые задачи могут использовать дополнительные потоки.
- Долгие команды блокируют event loop: `KEYS`, большие `LRANGE`, большие Lua-скрипты, массовые удаления без `UNLINK`.
- Latency зависит не только от CPU, но и от сети, fork при persistence, memory fragmentation, slow commands, eviction.

## Структуры данных

### String

String - базовый тип: строка, число, сериализованный JSON/msgpack, token, lock value.

Команды:

- `GET`, `SET`, `MGET`, `MSET`
- `SET key value EX 60 NX`
- `INCR`, `DECR`, `INCRBY`, `INCRBYFLOAT`
- `GETDEL`, `GETEX`

Типичные применения:

- кеш одного объекта;
- counters;
- feature flags;
- idempotency keys;
- distributed lock token.

Pitfall: хранить большие JSON-объекты удобно, но partial update невозможен без перезаписи всего значения. Для больших значений растут сетевые расходы и latency.

### Hash

Hash - map полей внутри одного ключа.

Команды:

- `HGET`, `HSET`, `HMGET`, `HGETALL`
- `HINCRBY`, `HDEL`, `HEXISTS`

Применения:

- профиль пользователя;
- состояние entity;
- компактное хранение множества мелких полей.

Pitfall: TTL ставится на весь hash-key, не на отдельное поле. Если нужен TTL на поле, нужны отдельные ключи или своя логика.

### List

List - linked list / quicklist, удобен для очередей и стеков.

Команды:

- `LPUSH`, `RPUSH`, `LPOP`, `RPOP`
- `BLPOP`, `BRPOP`
- `LTRIM`

Применения:

- простая очередь;
- recent items;
- bounded log.

Pitfall: List не дает consumer groups, ack, replay и нормальную работу нескольких consumer-групп. Для этого лучше Streams.

### Set

Set - уникальные значения.

Команды:

- `SADD`, `SREM`, `SMEMBERS`, `SISMEMBER`
- `SUNION`, `SINTER`, `SDIFF`

Применения:

- уникальные пользователи;
- membership;
- теги;
- антидублирование.

Pitfall: `SMEMBERS` на большом set может заблокировать Redis и сеть. Для обхода использовать `SSCAN`.

### Sorted Set

Sorted Set - уникальные значения с score.

Команды:

- `ZADD`, `ZREM`, `ZRANGE`, `ZREVRANGE`
- `ZRANGEBYSCORE`, `ZCOUNT`
- `ZPOPMIN`, `ZPOPMAX`

Применения:

- leaderboard;
- delayed queue;
- time-series index;
- rate limiting sliding window;
- ranking.

Pitfall: score - double. Для точности денег и строгих integer-сценариев лучше хранить integer score, например timestamp в миллисекундах.

### Bitmap

Bitmap - операции над битами внутри string.

Команды:

- `SETBIT`, `GETBIT`, `BITCOUNT`, `BITOP`

Применения:

- daily active users;
- flags;
- compact boolean membership.

Pitfall: большой offset расширяет строку до нужного размера. Ошибка в user id может внезапно съесть память.

### HyperLogLog

Приблизительный подсчет cardinality.

Команды:

- `PFADD`, `PFCOUNT`, `PFMERGE`

Применения:

- уникальные посетители;
- approximate analytics.

Pitfall: результат приблизительный, не использовать для billing или строгих отчетов.

### Geospatial

Geo хранится поверх sorted set.

Команды:

- `GEOADD`, `GEOPOS`, `GEOSEARCH`, `GEODIST`

Применения:

- поиск объектов рядом;
- delivery zones;
- nearest shops.

Pitfall: это не полноценная GIS-БД. Для сложной геометрии нужен PostGIS/Elasticsearch/OpenSearch.

### Stream

Stream - append-only log с id записей, consumer groups, ack и replay.

Команды:

- `XADD`, `XREAD`, `XRANGE`, `XREVRANGE`
- `XGROUP CREATE`, `XREADGROUP`
- `XACK`, `XPENDING`, `XAUTOCLAIM`
- `XTRIM`

Применения:

- event stream внутри системы;
- надежнее, чем pub/sub, для async processing;
- fan-out через consumer groups;
- retry через pending entries list.

Pitfall: Redis Streams не Kafka. Они удобны для умеренной нагрузки и простых сценариев, но не заменяют Kafka при долгом retention, больших объемах, сложной replay-аналитике и независимых consumer offsets на масштабе.

## Кеширование

### Cache-aside

Самая частая стратегия:

1. Приложение читает из Redis.
2. При miss читает из БД.
3. Сохраняет результат в Redis с TTL.
4. Возвращает ответ.

Плюсы:

- просто;
- БД остается source of truth;
- легко внедрять постепенно.

Минусы:

- cache miss дает дополнительную задержку;
- возможен cache stampede;
- нужна явная invalidation.

Senior-ответ: cache-aside - default choice для PHP backend, но нужен дизайн ключей, TTL с jitter, защита от dogpile, observability hit ratio и стратегия invalidation.

### Read-through

Кеш сам знает, как загрузить данные. В Redis напрямую это обычно реализуется на уровне библиотеки/сервиса, не самим Redis.

Плюсы:

- единый слой загрузки;
- меньше дублирования в коде.

Минусы:

- сложнее контролировать ошибки и fallback;
- tight coupling между cache layer и data source.

### Write-through

Запись идет одновременно в кеш и БД синхронно.

Плюсы:

- после записи кеш актуален;
- меньше stale reads.

Минусы:

- запись медленнее;
- нужно продумать атомарность между Redis и БД;
- partial failure приводит к расхождению.

### Write-behind / write-back

Запись сначала в кеш/очередь, потом асинхронно в БД.

Плюсы:

- высокая throughput;
- разгрузка БД.

Минусы:

- риск потери данных;
- сложная доставка, retry, idempotency;
- сложная операционная поддержка.

Senior-ответ: для критичных данных write-behind требует durable queue, idempotent consumers, мониторинг lag и понятную стратегию восстановления.

## Дизайн ключей

Хороший ключ должен быть читаемым, стабильным и не слишком длинным.

Примеры:

```text
user:{123}:profile
product:{42}:price
tenant:{7}:permissions:user:{123}
api:v2:catalog:category:{15}:page:{2}:sort:popular
lock:invoice:{987}
idempotency:payment:{request_id}
```

Рекомендации:

- Использовать namespace и версию: `app:v1:...`.
- Включать tenant id в multi-tenant системах.
- Для Redis Cluster использовать hash tags, если нужны multi-key операции в одном slot: `user:{123}:profile`, `user:{123}:permissions`.
- Не использовать `KEYS` в production, использовать `SCAN`.
- Не делать ключи на основе нестабильного JSON без нормализации параметров.

## TTL

TTL ограничивает stale data и память.

Команды:

- `EXPIRE`, `PEXPIRE`
- `TTL`, `PTTL`
- `PERSIST`
- `SET key value EX 60`

Практики:

- Почти все кеш-ключи должны иметь TTL.
- Добавлять jitter: `base_ttl + random(0, 10%)`, чтобы не истекали тысячи ключей одновременно.
- Разные TTL для разных типов данных: справочники дольше, персональные данные короче.
- Negative caching для отсутствующих данных, но с коротким TTL.
- TTL не заменяет invalidation, если нужна свежесть после записи.

Pitfall: слишком короткий TTL может создать постоянную нагрузку на БД. Слишком длинный TTL может закрепить баг или stale data.

## Invalidation

There are only two hard things in Computer Science: cache invalidation and naming things.

Стратегии invalidation:

- Delete on write: после успешной записи в БД удалить кеш.
- Update on write: после записи обновить кеш новым значением.
- Versioned keys: менять версию namespace или entity version.
- Tag-based invalidation: хранить связи tag -> keys, но нужно обслуживать рост set/list.
- Event-driven invalidation: доменное событие после commit транзакции БД удаляет/обновляет кеш.
- Short TTL: принять eventual consistency.

Правильный порядок при cache-aside:

1. Записать в БД.
2. После commit удалить или обновить кеш.
3. Если invalidation асинхронная, обеспечить retry и мониторинг.

Pitfall: удалить кеш до commit БД - можно получить race condition, где другой запрос прочитает старую БД и снова положит старые данные в кеш.

Senior-ответ: для большинства CRUD-сценариев безопаснее delete-after-commit плюс TTL как страховка. Для горячих данных можно обновлять кеш, но нужно контролировать race conditions.

## Eviction policies

Redis начинает вытеснять ключи, когда достигнут `maxmemory`, если настроена policy.

Основные политики:

- `noeviction` - не вытеснять, команды записи получают ошибку.
- `allkeys-lru` - вытеснять примерно least recently used среди всех ключей.
- `volatile-lru` - LRU только среди ключей с TTL.
- `allkeys-lfu` - least frequently used среди всех ключей.
- `volatile-lfu` - LFU только среди ключей с TTL.
- `allkeys-random` - случайные ключи.
- `volatile-random` - случайные ключи с TTL.
- `volatile-ttl` - ключи с ближайшим TTL.

Практический выбор:

- Для чистого cache Redis часто выбирают `allkeys-lru` или `allkeys-lfu`.
- Для Redis, где есть не только кеш, осторожнее: `noeviction` или отдельные инстансы под разные workloads.
- Не смешивать critical state и disposable cache в одном Redis без четкой memory policy.

Pitfall: eviction - не invalidation. Ключ может исчезнуть в любой момент, приложение должно корректно обрабатывать cache miss.

## Cache stampede / dogpile

Проблема: популярный ключ истек, много запросов одновременно идут в БД и пересчитывают значение.

Методы защиты:

- TTL jitter.
- Request coalescing: только один процесс пересчитывает, остальные ждут или получают stale.
- Mutex per key через `SET lock:key token NX PX 5000`.
- Stale-while-revalidate: отдавать старое значение и обновлять в фоне.
- Probabilistic early refresh: обновлять до истечения с вероятностью, зависящей от оставшегося TTL.
- Pre-warming для известных горячих ключей.
- Negative caching для частых miss по несуществующим объектам.

Пример dogpile lock:

```text
GET cache:key
if hit: return value

SET lock:cache:key token NX PX 5000
if lock acquired:
    value = load_from_db()
    SET cache:key value EX ttl_with_jitter
    delete lock only if token matches
else:
    sleep small jitter
    retry GET cache:key
    if still miss: fallback to DB with rate limit or return stale
```

Pitfall: lock TTL должен быть больше ожидаемого времени генерации, но не бесконечным. Unlock должен проверять token, иначе один процесс может удалить чужой lock.

## Distributed locks

Минимальный lock на одном Redis:

```text
SET lock:resource unique_token NX PX 10000
```

Unlock через Lua:

```lua
if redis.call("GET", KEYS[1]) == ARGV[1] then
    return redis.call("DEL", KEYS[1])
else
    return 0
end
```

Почему нужен token:

- Процесс A взял lock на 10 секунд.
- Процесс A завис на 15 секунд.
- Lock истек, процесс B взял новый lock.
- Процесс A проснулся и сделал `DEL lock`.
- Без проверки token A удалит lock B.

### Redlock

Redlock - алгоритм, где клиент пытается взять lock на большинстве независимых Redis master-инстансов и считает lock успешным, если уложился во время lease.

Важные caveats:

- Redlock не дает линейризуемую координацию в строгом смысле при всех сетевых partition и clock drift.
- Для критичных операций, где двойное выполнение недопустимо, лучше использовать БД с unique constraint / advisory lock, etcd, ZooKeeper, Consul или fencing tokens.
- Даже с lock нужна идемпотентность операции.
- Для внешних side effects нужен fencing token: монотонный номер, который downstream проверяет и отвергает старые операции.

Senior-ответ: Redis lock подходит для best-effort защиты от параллельной работы, dogpile prevention, scheduled jobs. Для финансовых переводов и эксклюзивного доступа к критическому ресурсу одного lock в Redis недостаточно; нужны транзакции в source of truth, idempotency и fencing.

## Lua scripts

Lua в Redis позволяет выполнить несколько операций атомарно на одном инстансе.

Команды:

- `EVAL`
- `EVALSHA`
- `SCRIPT LOAD`
- `SCRIPT EXISTS`

Примеры применения:

- безопасный unlock;
- rate limiting;
- compare-and-set;
- атомарная проверка квоты и списание;
- move item между структурами.

Ограничения:

- Скрипт блокирует Redis на время выполнения.
- Нельзя делать долгие циклы по большим коллекциям.
- В Redis Cluster все ключи скрипта должны быть в одном hash slot.
- Нужно явно передавать ключи через `KEYS`, не генерировать произвольные key names внутри скрипта для Cluster.

Senior-ответ: Lua хорош для коротких атомарных операций рядом с данными. Если логика становится большой бизнес-процедурой, ее лучше вынести в приложение или БД.

## Transactions и pipelines

### Pipeline

Pipeline отправляет несколько команд без ожидания ответа после каждой. Это снижает round-trip latency.

Свойства:

- Не атомарен.
- Команды выполняются последовательно, но другие клиенты могут вклиниться между ними.
- Хорош для batch GET/SET, counters, массовых операций.

### Transaction MULTI/EXEC

Redis transaction:

```text
MULTI
SET a 1
INCR counter
EXEC
```

Свойства:

- Команды после `MULTI` ставятся в очередь.
- `EXEC` выполняет их последовательно без interleaving команд других клиентов.
- Нет rollback как в SQL.
- Ошибки команд могут проявиться уже внутри результата `EXEC`.

### WATCH

Optimistic locking:

```text
WATCH key
value = GET key
MULTI
SET key new_value
EXEC
```

Если key изменился между `WATCH` и `EXEC`, transaction abort.

Pitfall: при высокой конкуренции `WATCH` может часто конфликтовать. Иногда Lua проще и надежнее.

## Pub/Sub vs Streams

### Pub/Sub

Pub/Sub - fire-and-forget messaging.

Свойства:

- Нет persistence сообщений.
- Если subscriber offline, он пропускает сообщения.
- Нет ack, retry, consumer groups.
- Хорош для live notifications, invalidation hints, websocket fan-out.

Pitfall: использовать Pub/Sub как надежную очередь - ошибка.

### Streams

Streams - persistent log в Redis memory/persistence model.

Свойства:

- Сообщения имеют id.
- Можно читать с конкретного id.
- Consumer groups распределяют сообщения между consumers.
- Есть pending entries list и ack.
- Можно reclaim stuck messages.

Хорошо подходит для:

- фоновой обработки;
- интеграции между сервисами при умеренном объеме;
- retryable jobs;
- event-driven invalidation.

## Streams и consumer groups

Создание stream/group:

```text
XGROUP CREATE orders group-1 $ MKSTREAM
```

Добавление события:

```text
XADD orders * type created order_id 123 user_id 77
```

Чтение группой:

```text
XREADGROUP GROUP group-1 consumer-1 COUNT 10 BLOCK 5000 STREAMS orders >
```

Ack:

```text
XACK orders group-1 1710000000000-0
```

Диагностика pending:

```text
XPENDING orders group-1
```

Reclaim:

```text
XAUTOCLAIM orders group-1 consumer-2 60000 0-0 COUNT 10
```

Trim:

```text
XTRIM orders MAXLEN ~ 100000
```

Практики:

- Делать обработчики идемпотентными.
- Ack только после успешной обработки.
- Иметь retry policy и dead-letter stream.
- Мониторить lag, pending count, oldest pending age.
- Ограничивать stream size через `XTRIM`, иначе память будет расти.

Pitfall: если consumer умер после side effect, но до `XACK`, событие будет обработано повторно. Поэтому нужна идемпотентность.

## Persistence: RDB и AOF

Redis может сохранять данные на диск, но persistence не делает его автоматически полноценной durable БД.

### RDB

RDB - snapshot данных в определенные моменты.

Плюсы:

- компактный файл;
- быстрый restart по сравнению с большим AOF;
- удобен для backup.

Минусы:

- можно потерять данные после последнего snapshot;
- fork может создать нагрузку на память из-за copy-on-write;
- snapshot больших данных может влиять на latency.

### AOF

AOF - append-only file с логом write-команд.

Настройки fsync:

- `appendfsync always` - надежнее, но медленнее.
- `appendfsync everysec` - типичный компромисс, потеря до 1 секунды.
- `appendfsync no` - быстрее, но зависит от OS flush.

Плюсы:

- меньше потеря данных;
- понятный лог команд;
- rewrite уменьшает размер.

Минусы:

- больше I/O;
- AOF rewrite тоже может влиять на latency;
- при `everysec` все равно есть окно потери.

### RDB + AOF

Частый production-вариант - использовать оба механизма. RDB удобен для snapshot/backup, AOF уменьшает окно потери.

Senior-ответ: если Redis используется только как кеш, persistence часто отключают или минимизируют. Если Redis хранит состояние очередей/streams/locks/counters, persistence надо проектировать осознанно, с пониманием допустимого RPO/RTO.

## Replication

Redis replication обычно async: master принимает запись, replicas догоняют.

Свойства:

- Read replicas могут отдавать stale data.
- При failover возможна потеря последних записей.
- Replica может использоваться для read scaling, backup, failover.
- `WAIT` может повысить уверенность, что запись дошла до replicas, но не превращает Redis в CP-систему.

Pitfall: после failover старый master мог принять запись, которая не попала на promoted replica. Приложение должно понимать eventual consistency.

## Sentinel

Sentinel обеспечивает high availability для Redis master-replica:

- мониторит master и replicas;
- выбирает новую master при отказе;
- уведомляет клиентов о новом master;
- требует кворум Sentinel-инстансов.

Важные моменты:

- Sentinel не шардирует данные.
- Клиент должен поддерживать Sentinel discovery.
- Failover не мгновенный, возможны ошибки записи во время переключения.
- Async replication означает возможную потерю последних записей.

Senior-ответ: Sentinel - HA для одного набора данных. Если нужен horizontal scaling по памяти/throughput, нужен Cluster или application-level sharding.

## Redis Cluster

Redis Cluster делит keyspace на 16384 hash slots. Каждый master отвечает за часть slots, replicas обеспечивают failover.

Особенности:

- Sharding по key hash slot.
- Клиент должен понимать `MOVED` и `ASK` redirects.
- Multi-key операции работают только для ключей в одном slot.
- Hash tags позволяют управлять slot: `cart:{user123}:items`, `cart:{user123}:meta`.
- Cluster не поддерживает несколько баз данных как standalone Redis: используется DB 0.

Что усложняется:

- `MGET`/`MSET` по ключам из разных slots.
- Lua scripts с несколькими ключами.
- Transactions по ключам из разных slots.
- Глобальный scan/analytics.
- Client library configuration и failover behavior.

Senior-ответ: Cluster стоит вводить, когда один Redis не хватает по памяти/throughput или нужен managed horizontal scaling. Не стоит вводить Cluster только "на всякий случай", потому что он усложняет ключи, операции и эксплуатацию.

## Memory diagnostics

Команды и инструменты:

- `INFO memory` - used memory, RSS, fragmentation, allocator stats.
- `INFO stats` - hits/misses, evicted keys, expired keys.
- `INFO commandstats` - статистика команд.
- `MEMORY USAGE key` - оценка памяти ключа.
- `MEMORY STATS` - подробная память.
- `MEMORY DOCTOR` - диагностические рекомендации.
- `SLOWLOG GET` - медленные команды.
- `LATENCY DOCTOR` - диагностика latency events.
- `SCAN` вместо `KEYS`.
- `redis-cli --bigkeys` - поиск больших ключей.
- `redis-cli --memkeys` - анализ ключей по памяти.

Метрики, которые стоит мониторить:

- memory usage и процент от `maxmemory`;
- fragmentation ratio;
- evicted keys;
- expired keys;
- keyspace hits/misses;
- blocked clients;
- connected clients;
- instantaneous ops/sec;
- replication lag;
- AOF/RDB status;
- slowlog;
- stream pending/lag.

Pitfall: высокий RSS при умеренном used memory может быть fragmentation или copy-on-write после fork. Не всегда помогает `DEL`; иногда нужен active defrag или restart replica/failover.

## PHP integration notes

### Клиенты

Популярные варианты:

- `ext-redis` / PhpRedis - C extension, высокая производительность, широко используется.
- Predis - pure PHP client, проще ставить, но обычно медленнее.
- Symfony Cache Redis adapter.
- Laravel Redis через PhpRedis или Predis.

Senior-подход:

- Для high-load PHP чаще выбирать PhpRedis.
- Настроить persistent connections осознанно: они могут помочь, но при неправильной настройке создают проблемы с connection state.
- Учитывать FPM model: много воркеров, каждый может держать соединения.
- Настраивать timeouts: connect timeout, read timeout, retry/backoff.
- Не делать Redis безлимитным single point of failure для request path.

### Serialization

Варианты:

- PHP serialize - просто, но привязано к PHP-классам.
- igbinary - компактнее и быстрее, если доступен extension.
- JSON - межъязыково, но теряет типы и может быть больше.
- msgpack - компактно и межъязыково, но нужен extension/library.

Pitfall: кешировать PHP objects опасно при изменении классов между релизами. Для долгоживущего кеша лучше DTO array/json/schema-version.

### Laravel/Symfony

Laravel:

- cache store Redis;
- queue driver Redis;
- locks через cache lock API;
- Horizon для очередей;
- важно разделять prefix/database/connection для cache, queue, sessions.

Symfony:

- Cache component RedisAdapter;
- Lock component с RedisStore;
- Messenger transport через Redis Streams;
- важно настроить namespace, marshalling и stampede protection.

### Ошибки интеграции

- Не заданы timeouts, PHP-запросы висят на Redis.
- Один Redis используется для cache, sessions, queues, locks без изоляции.
- Нет fallback при Redis outage.
- Lock без token-safe unlock.
- Cache keys не версионируются, после релиза старые payload ломают код.
- Большие `MGET`/pipeline без ограничения batch size.
- Использование `KEYS` в production.
- Хранение пользовательских сессий в Redis без persistence/replication там, где logout всех пользователей неприемлем.

## Типовые pitfalls

- Считать Redis durable БД без анализа RDB/AOF и failover windows.
- Считать Pub/Sub надежной очередью.
- Использовать lock без TTL.
- Удалять lock без проверки token.
- Не добавлять TTL к cache keys.
- Делать массовое истечение ключей в одну секунду без jitter.
- Использовать `KEYS *` на production.
- Хранить огромные значения и читать их на каждый request.
- Смешивать разные workloads в одном Redis без memory policy.
- Не мониторить evictions и hit ratio.
- Делать multi-key операции в Cluster без hash tags.
- Использовать replicas для чтения строго консистентных данных.
- Не делать stream consumers идемпотентными.
- Не чистить streams через trim.
- Писать длинные Lua-скрипты, блокирующие Redis.

## Senior answers

### Когда Redis хорош как кеш?

Когда данные часто читаются, дороже получать их из БД/API, допустима eventual consistency, есть понятная invalidation или TTL, а приложение умеет обрабатывать cache miss. Я бы начал с cache-aside, TTL+jitter, метрик hit ratio и защиты от stampede для горячих ключей.

### Как защититься от cache stampede?

Комбинировать TTL jitter, per-key lock через `SET NX PX`, stale-while-revalidate для горячих данных и ограничение параллельных rebuild. Lock должен иметь token и безопасный Lua-unlock. Для особо горячих данных можно делать pre-warm и probabilistic early refresh.

### Redis lock безопасен?

Для best-effort coordination - да, если `SET NX PX`, unique token и Lua unlock. Для критичных операций - недостаточно. Нужны идемпотентность, fencing tokens или lock/constraint в source of truth. Redlock имеет известные caveats и не заменяет consensus-системы.

### Чем Pub/Sub отличается от Streams?

Pub/Sub доставляет только online subscribers, без persistence, ack и retry. Streams хранят log, позволяют consumer groups, pending entries, ack, replay и reclaim. Для надежной фоновой обработки лучше Streams, для live notifications можно Pub/Sub.

### RDB или AOF?

RDB - snapshots, быстрее и компактнее, но можно потерять изменения после snapshot. AOF пишет лог операций, обычно с `appendfsync everysec`, снижает потерю до примерно секунды, но дает больше I/O. Для кеша persistence может быть не нужна, для streams/queues/counters нужно выбирать по RPO/RTO.

### Sentinel или Cluster?

Sentinel дает HA для одного master-replica набора, без sharding. Cluster дает sharding по hash slots и failover, но усложняет multi-key операции, scripts и клиентскую логику. Если проблема только в failover - Sentinel. Если не хватает памяти/throughput одного master - Cluster.

### Как диагностировать память?

Начал бы с `INFO memory`, `INFO stats`, evicted keys, fragmentation ratio, `MEMORY STATS`, `MEMORY USAGE` для подозрительных ключей, `redis-cli --bigkeys/--memkeys`, slowlog и latency doctor. Дальше проверил бы TTL coverage, большие коллекции, stream trimming, eviction policy и разделение workloads.

## Self-check

Ответьте без подсказок:

- Почему `SETNX` без TTL опасен?
- Зачем lock token и Lua unlock?
- Чем `allkeys-lru` отличается от `volatile-lru`?
- Почему `KEYS *` опасен?
- Что произойдет с Pub/Sub сообщением, если subscriber offline?
- Что такое pending entries list в Streams?
- Почему stream consumer должен быть идемпотентным?
- Как hash tags помогают в Redis Cluster?
- Почему Redis replica может вернуть stale data?
- Какой RPO у AOF `everysec`?
- Чем pipeline отличается от transaction?
- Почему `MULTI/EXEC` не дает rollback как SQL?
- Когда `WATCH` лучше заменить Lua-скриптом?
- Какие метрики покажут, что кеш работает плохо?
- Почему нельзя полагаться только на TTL для invalidation после write?

## Mini-practice

### Задача 1: кеш профиля пользователя

Спроектируйте cache-aside для `GET /users/{id}`:

- ключ;
- TTL;
- negative caching;
- invalidation после update;
- защита от stampede.

Ожидаемый senior-ответ:

- `user:{id}:profile:v1`, TTL 5-30 минут с jitter;
- miss -> DB -> set cache;
- not found кешировать на 30-60 секунд;
- after DB commit delete cache;
- для hot users использовать lock `lock:user:{id}:profile` и stale fallback.

### Задача 2: distributed scheduled job

Нужно, чтобы cron-задача не запускалась параллельно на нескольких PHP-инстансах.

Решение:

- lock key `lock:job:{name}`;
- `SET key token NX PX job_timeout`;
- job должен быть идемпотентным;
- lock продлевать только если job точно жив;
- unlock через Lua token check;
- для критичных задач использовать DB advisory lock или fencing.

### Задача 3: очередь событий на Streams

Спроектируйте обработку `order.created`:

- stream `orders.events`;
- group `billing-service`;
- consumer per worker;
- `XREADGROUP ... >`;
- `XACK` после успешной обработки;
- retry через `XPENDING`/`XAUTOCLAIM`;
- dead-letter stream после N попыток;
- idempotency key по event id/order id.

### Задача 4: memory incident

Redis начал evict keys, hit ratio упал, latency выросла.

План диагностики:

- проверить `INFO memory`, `INFO stats`, `evicted_keys`;
- найти большие ключи через `--bigkeys`/`--memkeys`;
- проверить TTL coverage и stream/list sizes;
- посмотреть slowlog и latency doctor;
- проверить maxmemory-policy;
- разделить cache/session/queue workloads при необходимости;
- добавить trim, TTL, batch limits, compression или изменить eviction policy.

## Checklist перед production

- Есть понятная роль Redis: cache, session, queue, stream, lock или state.
- Workloads разделены или хотя бы изолированы prefix/database/cluster.
- Настроены `maxmemory` и eviction policy.
- У cache keys есть TTL и jitter.
- Есть invalidation после write.
- Есть защита от stampede для hot keys.
- Locks используют `SET NX PX`, unique token и Lua unlock.
- Операции под lock идемпотентны.
- Lua-скрипты короткие и не обходят большие коллекции.
- Pipeline batch size ограничен.
- Нет `KEYS` в production path.
- Streams имеют trim, retry, DLQ и idempotent consumers.
- Persistence соответствует RPO/RTO.
- Replication/Sentinel/Cluster протестированы на failover.
- PHP-клиенты имеют timeouts и retry policy.
- Сериализация совместима с релизами.
- Мониторятся memory, hit ratio, evictions, latency, slowlog, replication lag, stream pending.
- Есть fallback/degradation при Redis outage.

## Быстрые формулы для интервью

- Cache-aside: app controls cache, DB source of truth.
- TTL without jitter causes synchronized expiration.
- Eviction is not invalidation.
- Pub/Sub is not a queue.
- Streams need idempotent consumers.
- Redis lock is a lease, not eternal ownership.
- Redlock is not a silver bullet for correctness.
- AOF everysec can still lose about one second.
- Replication is async unless explicitly designed around `WAIT`, and even then not full consensus.
- Cluster scales keyspace, but breaks cross-slot assumptions.

## Ссылки

- Redis Docs: Data types - https://redis.io/docs/latest/develop/data-types/
- Redis Docs: Commands - https://redis.io/docs/latest/commands/
- Redis Docs: Expiration - https://redis.io/docs/latest/commands/expire/
- Redis Docs: Eviction policies - https://redis.io/docs/latest/develop/reference/eviction/
- Redis Docs: Pipelining - https://redis.io/docs/latest/develop/using-commands/pipelining/
- Redis Docs: Transactions - https://redis.io/docs/latest/develop/using-commands/transactions/
- Redis Docs: Programmability and Lua - https://redis.io/docs/latest/develop/programmability/eval-intro/
- Redis Docs: Distributed locks with Redis - https://redis.io/docs/latest/develop/use/patterns/distributed-locks/
- Redis Docs: Streams - https://redis.io/docs/latest/develop/data-types/streams/
- Redis Docs: Persistence - https://redis.io/docs/latest/operate/oss_and_stack/management/persistence/
- Redis Docs: Replication - https://redis.io/docs/latest/operate/oss_and_stack/management/replication/
- Redis Docs: Sentinel - https://redis.io/docs/latest/operate/oss_and_stack/management/sentinel/
- Redis Docs: Cluster tutorial - https://redis.io/docs/latest/operate/oss_and_stack/management/scaling/
- Redis Docs: Memory optimization - https://redis.io/docs/latest/operate/oss_and_stack/management/optimization/memory-optimization/
- Redis Docs: Observability - https://redis.io/docs/latest/operate/oss_and_stack/management/optimization/observability/
- Martin Kleppmann: How to do distributed locking - https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html
- Antirez: Is Redlock safe? - http://antirez.com/news/101
- AWS Database Blog: Database caching strategies using Redis - https://aws.amazon.com/caching/database-caching/
- Cloudflare Blog: Stale-while-revalidate and cache stampede context - https://blog.cloudflare.com/sometimes-i-cache/
