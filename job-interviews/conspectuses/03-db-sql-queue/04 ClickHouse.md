# ClickHouse: MergeTree, materialized views, ingestion, distributed tables

Гайд для Lead/Senior PHP backend developer.

Фокус: когда выбирать ClickHouse, как проектировать таблицы под аналитические запросы,
какие trade-off'ы важно проговаривать на интервью и где ClickHouse не должен заменять OLTP-БД.

Материал собран из исходных конспектов:

- `PostgreSQL ClickHouse Redis RabbitMQ, индексы, транзакции, блокировки.md`
- `Масштабирование баз данных партиционирование репликация шардирование.md`

## Когда использовать

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

## MergeTree Basics

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

## PARTITION BY, ORDER BY, PRIMARY KEY

`PARTITION BY` нужен не для ускорения каждого запроса, а для lifecycle операций:
drop partition, TTL, управление объемом parts.

Хороший partition key:

- не слишком мелкий;
- обычно месяц/день для time-series;
- соответствует retention и загрузке.

Ловушка: `PARTITION BY user_id` при миллионах пользователей создаст слишком много partitions и parts.

`ORDER BY` выбирается под самые частые фильтры и группировки.
Колонки с высокой селективностью и частым фильтром обычно ставят раньше,
но нужно учитывать реальные запросы.

Senior-ответ: в ClickHouse `PRIMARY KEY` не гарантирует уникальность. Это sparse index для ускорения чтения.

## Материализованные представления

Materialized View в ClickHouse обычно используется как insert-trigger:
при вставке в source table данные трансформируются и пишутся в target table.

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

## Ingestion

Рекомендации:

- вставлять батчами, а не по одной строке;
- избегать слишком большого количества small inserts;
- использовать Kafka engine/Buffer/async inserts при необходимости;
- следить за количеством parts;
- проектировать schema под запросы, а не нормализовать как OLTP.

PHP-практика: для событий лучше буферизовать и писать пачками через worker,
чем синхронно вставлять каждое событие из web request.

## Когда не использовать ClickHouse

Не стоит выбирать ClickHouse, если нужны:

- банковские транзакции с сильной консистентностью;
- частые `UPDATE`/`DELETE` по одной строке;
- сложные FK и constraints;
- основной источник истины для mutable бизнес-объектов;
- низкая задержка для большого числа point lookups по PK.

## Distributed tables и replication basics

ClickHouse чаще используют для аналитики, логов, событий и агрегатов, а не как primary OLTP DB.

Основные сущности:

- `MergeTree`: базовое семейство движков для хранения.
- `ReplicatedMergeTree`: репликация таблицы между нодами.
- `Distributed`: logical table, которая распределяет запросы по shards/replicas.
- Shard: часть данных.
- Replica: копия shard для HA/read scaling.

Как работает на практике:

- Данные часто пишут в локальные `ReplicatedMergeTree` таблицы.
- `Distributed` таблица используется как entrypoint для запросов или insert routing.
- Для cluster metadata и coordination традиционно используется ZooKeeper или ClickHouse Keeper.

Trade-offs:

- Отличен для больших append-only аналитических объемов.
- Не замена PostgreSQL для сложных OLTP транзакций.
- Mutations/updates/deletes дорогие относительно append/read сценариев.
- Нужно проектировать partition key, order by, primary key под запросы.
- Distributed queries могут быть тяжелыми, если не фильтровать по shard/partition-friendly ключам.

Короткий senior-ответ:

> ClickHouse масштабирует аналитику через shards и replicas.
> `Distributed` таблица не хранит данные сама как обычная OLTP-таблица,
> а маршрутизирует запросы к локальным таблицам кластера.
> Схему надо проектировать под append и аналитические выборки.

## ClickHouse в system design

ClickHouse подходит как специализированная read-модель для аналитики, логов, событий и cross-shard analytics.

Типичные сценарии:

- вынести аналитику из OLTP PostgreSQL;
- агрегировать события из outbox/event stream;
- хранить события для отчетов, где допустима задержка;
- использовать как analytical store для scatter-gather данных из shards.

Что проговорить на интервью:

- ClickHouse не заменяет source of truth для mutable бизнес-объектов.
- Схему проектируют под главные запросы, а не по нормальной форме.
- Insert должен идти батчами.
- `PARTITION BY` отвечает за lifecycle, `ORDER BY` - за физический порядок и data skipping.
- Materialized views не решают backfill автоматически.
- Distributed queries и cluster topology нужно проектировать отдельно.

## Короткие ответы

**Как выбрать `ORDER BY` в ClickHouse?**

По основным фильтрам и группировкам запросов.
`ORDER BY` задает физическую сортировку и sparse index, поэтому от него зависит data skipping.
Нельзя выбирать только по красоте схемы.

**Почему ClickHouse `PRIMARY KEY` не гарантирует уникальность?**

Это sparse index для чтения, а не constraint уникальности.

**Чем `PARTITION BY` отличается от `ORDER BY` в MergeTree?**

`PARTITION BY` управляет разбиением parts/lifecycle, `ORDER BY` задает физическую сортировку и data skipping.

**Почему materialized view в ClickHouse не решает backfill автоматически?**

MV срабатывает на новые вставки; старые данные нужно переливать отдельно.

## Checklist

- Понятны топ-5 запросов.
- `ORDER BY` выбран под фильтрацию и агрегации.
- `PARTITION BY` не создает тысячи мелких partitions.
- Insert идет батчами.
- Есть стратегия retention/TTL.
- Backfill для MV продуман отдельно.
- Понятно, где source of truth, а где analytical read-model.
- Distributed queries фильтруются по shard/partition-friendly ключам, если это возможно.
- Cluster metadata и coordination имеют понятную operational-модель.

## Вопросы для самопроверки

1. Когда ClickHouse лучше PostgreSQL для аналитического запроса?
2. Почему ClickHouse плохо подходит для OLTP с частыми update/delete?
3. Чем `PARTITION BY` отличается от `ORDER BY`?
4. Почему `PRIMARY KEY` в ClickHouse не означает уникальность?
5. Что такое part и почему small inserts могут быть проблемой?
6. Почему materialized view не пересчитывает старые данные автоматически?
7. Когда нужен `Distributed` engine?
8. Чем shard отличается от replica?
9. Почему distributed queries могут быть дорогими?
10. Как использовать ClickHouse для cross-shard analytics?

## Ссылки

- [ClickHouse Documentation](https://clickhouse.com/docs)
- [ClickHouse MergeTree](https://clickhouse.com/docs/engines/table-engines/mergetree-family/mergetree)
- [ClickHouse Materialized Views](https://clickhouse.com/docs/materialized-view)
- [ClickHouse Best Practices](https://clickhouse.com/docs/best-practices)
- [ClickHouse Docs: Distributed table engine](https://clickhouse.com/docs/engines/table-engines/special/distributed)
- [ClickHouse Docs: Replication](https://clickhouse.com/docs/engines/table-engines/mergetree-family/replication)
- [ClickHouse Docs: ClickHouse Keeper](https://clickhouse.com/docs/guides/sre/keeper/clickhouse-keeper)
- [ClickHouse Documentation: SELECT](https://clickhouse.com/docs/sql-reference/statements/select)
- [ClickHouse Documentation: Window Functions](https://clickhouse.com/docs/sql-reference/window-functions)
