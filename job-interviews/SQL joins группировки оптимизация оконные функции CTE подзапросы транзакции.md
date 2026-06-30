# SQL: JOIN, группировки, оптимизация, оконные функции, CTE, подзапросы, транзакции

Конспект для Lead/Senior PHP backend developer, который готовится к интервью. Примеры в основном PostgreSQL-flavored. Где поведение заметно отличается в MySQL, ClickHouse или других СУБД, это отмечено отдельно.

## Карта тем

- Логический порядок выполнения `SELECT`.
- Типы соединений: `INNER`, `LEFT`, `RIGHT`, `FULL`, `CROSS`, semi/anti join.
- Кардинальность, дубликаты, `NULL`-семантика.
- `GROUP BY`, `HAVING`, агрегаты.
- Основы оптимизации, индексы, sargability, `EXPLAIN`.
- Оконные функции: `OVER`, `PARTITION BY`, `ORDER BY`, frames, ranking, running totals.
- CTE, recursive CTE, материализация.
- Подзапросы: коррелированные, некоррелированные, `EXISTS` vs `IN`.
- Транзакции: ACID, уровни изоляции, блокировки, deadlocks, MVCC.

## Логический порядок выполнения SELECT

Важно отличать логический порядок от физического плана выполнения. Оптимизатор может переставлять операции, но смысл запроса описывается примерно так:

1. `WITH` - подготовка CTE.
2. `FROM` - выбор источников.
3. `JOIN ... ON` - соединения и условия соединения.
4. `WHERE` - фильтрация строк до группировки.
5. `GROUP BY` - группировка.
6. Агрегатные функции - расчет агрегатов по группам.
7. `HAVING` - фильтрация групп.
8. Оконные функции - расчет поверх результирующих строк после `WHERE`/`GROUP BY`/`HAVING`.
9. `SELECT` - формирование выражений результата.
10. `DISTINCT` - удаление дублей.
11. `ORDER BY` - сортировка результата.
12. `LIMIT`/`OFFSET`/`FETCH` - ограничение выдачи.

Практическое объяснение на интервью: `WHERE` не видит алиасы из `SELECT` и не может фильтровать по агрегатам; для агрегатов нужен `HAVING`, а для оконных функций обычно нужен внешний запрос или CTE.

```sql
SELECT customer_id, total
FROM (
    SELECT
        customer_id,
        SUM(amount) AS total,
        RANK() OVER (ORDER BY SUM(amount) DESC) AS rnk
    FROM orders
    WHERE status = 'paid'
    GROUP BY customer_id
) t
WHERE rnk <= 10;
```

## JOIN: базовые типы

### INNER JOIN

Возвращает только строки, для которых нашлась пара в обеих таблицах.

```sql
SELECT o.id, c.email
FROM orders o
INNER JOIN customers c ON c.id = o.customer_id;
```

Если `orders.customer_id` не имеет соответствующего `customers.id`, заказ не попадет в результат.

### LEFT JOIN

Возвращает все строки из левой таблицы и совпадения из правой. Если совпадения нет, поля правой таблицы будут `NULL`.

```sql
SELECT c.id, c.email, o.id AS order_id
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id;
```

Типичная ошибка: условие по правой таблице в `WHERE` превращает `LEFT JOIN` в фактический `INNER JOIN`.

```sql
-- Ошибка, клиенты без заказов исчезнут
SELECT c.id, o.id
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id
WHERE o.status = 'paid';

-- Правильно, если нужно сохранить клиентов без paid-заказов
SELECT c.id, o.id
FROM customers c
LEFT JOIN orders o
    ON o.customer_id = c.id
   AND o.status = 'paid';
```

### RIGHT JOIN

Зеркальный `LEFT JOIN`: сохраняет все строки из правой таблицы. В продакшене часто заменяют на `LEFT JOIN`, поменяв таблицы местами, потому что так проще читать.

```sql
SELECT o.id, c.email
FROM orders o
RIGHT JOIN customers c ON c.id = o.customer_id;
```

### FULL OUTER JOIN

Возвращает все строки из обеих таблиц: совпавшие объединяются, несовпавшие дополняются `NULL`.

```sql
SELECT a.external_id, a.value AS old_value, b.value AS new_value
FROM import_old a
FULL OUTER JOIN import_new b ON b.external_id = a.external_id;
```

Полезно для сверок. В MySQL `FULL OUTER JOIN` напрямую не поддерживается; часто эмулируется через `LEFT JOIN UNION RIGHT JOIN`.

### CROSS JOIN

Декартово произведение: каждая строка слева соединяется с каждой строкой справа.

```sql
SELECT d.day, s.slot
FROM days d
CROSS JOIN slots s;
```

Используется для генерации календарей, матриц, всех комбинаций. Опасен на больших таблицах.

## Semi Join и Anti Join

SQL обычно не имеет ключевых слов `SEMI JOIN`/`ANTI JOIN`, но оптимизатор может строить такие планы.

Semi join: вернуть строки из левой таблицы, для которых существует совпадение справа.

```sql
SELECT c.*
FROM customers c
WHERE EXISTS (
    SELECT 1
    FROM orders o
    WHERE o.customer_id = c.id
);
```

Anti join: вернуть строки из левой таблицы, для которых совпадения справа нет.

```sql
SELECT c.*
FROM customers c
WHERE NOT EXISTS (
    SELECT 1
    FROM orders o
    WHERE o.customer_id = c.id
);
```

Почему это лучше, чем `JOIN + DISTINCT`: не размножает строки клиента количеством заказов и явно выражает намерение.

## Дубликаты и кардинальность

Кардинальность соединения зависит от отношений между таблицами:

- `1:1` - одна строка к одной строке.
- `1:N` - одна строка слева может дать много строк результата.
- `N:M` - результат может расти взрывным образом, обычно через pivot/junction table.

Пример размножения строк:

```sql
SELECT c.id, c.email, o.id AS order_id
FROM customers c
JOIN orders o ON o.customer_id = c.id;
```

Если у клиента 10 заказов, клиент появится 10 раз. Это не баг `JOIN`, а ожидаемая кардинальность.

Для ответа на вопрос "есть ли хотя бы один заказ" используйте `EXISTS`, а не `JOIN DISTINCT`.

Для агрегации сначала часто выгодно агрегировать дочернюю таблицу, затем соединять:

```sql
SELECT c.id, c.email, s.paid_total
FROM customers c
LEFT JOIN (
    SELECT customer_id, SUM(amount) AS paid_total
    FROM orders
    WHERE status = 'paid'
    GROUP BY customer_id
) s ON s.customer_id = c.id;
```

## NULL-семантика

`NULL` означает неизвестное или отсутствующее значение. Сравнение с `NULL` через `=` не работает как ожидаемое булево сравнение.

```sql
SELECT * FROM users WHERE deleted_at = NULL;    -- неверно
SELECT * FROM users WHERE deleted_at IS NULL;   -- верно
SELECT * FROM users WHERE deleted_at IS NOT NULL;
```

Трехзначная логика SQL: выражение может быть `TRUE`, `FALSE`, `UNKNOWN`. В `WHERE` проходят только `TRUE`.

Особенности:

- `COUNT(*)` считает строки.
- `COUNT(column)` считает только не-`NULL` значения.
- `SUM`, `AVG`, `MIN`, `MAX` игнорируют `NULL`.
- `NULL IN (...)` и `NOT IN` с `NULL` часто дают неожиданный результат.

Опасный пример:

```sql
-- Если в подзапросе есть NULL, NOT IN может не вернуть ничего
SELECT c.*
FROM customers c
WHERE c.id NOT IN (SELECT customer_id FROM orders);
```

Надежнее:

```sql
SELECT c.*
FROM customers c
WHERE NOT EXISTS (
    SELECT 1
    FROM orders o
    WHERE o.customer_id = c.id
);
```

PostgreSQL поддерживает `IS DISTINCT FROM`, который сравнивает значения с учетом `NULL` как обычного различимого состояния.

```sql
SELECT *
FROM users
WHERE previous_email IS DISTINCT FROM email;
```

## GROUP BY, HAVING и агрегаты

`GROUP BY` сворачивает строки в группы. В `SELECT` можно указывать либо поля группировки, либо агрегатные выражения.

```sql
SELECT customer_id, COUNT(*) AS orders_count, SUM(amount) AS total_amount
FROM orders
WHERE status = 'paid'
GROUP BY customer_id
HAVING SUM(amount) > 1000;
```

`WHERE` фильтрует строки до группировки. `HAVING` фильтрует группы после агрегирования.

Полезные агрегаты PostgreSQL:

```sql
SELECT
    customer_id,
    COUNT(*) AS orders_count,
    COUNT(*) FILTER (WHERE status = 'paid') AS paid_orders_count,
    SUM(amount) FILTER (WHERE status = 'paid') AS paid_total,
    BOOL_OR(status = 'refunded') AS has_refund,
    ARRAY_AGG(id ORDER BY created_at DESC) AS order_ids,
    JSONB_AGG(JSONB_BUILD_OBJECT('id', id, 'amount', amount)) AS orders_json
FROM orders
GROUP BY customer_id;
```

Портативность:

- `FILTER (WHERE ...)` есть в PostgreSQL, но не в MySQL; в MySQL часто используют `SUM(CASE WHEN ... THEN 1 ELSE 0 END)`.
- `JSONB_AGG`, `ARRAY_AGG`, `BOOL_OR` зависят от СУБД.

Типичные ошибки:

- Группировать на неправильном уровне детализации.
- Использовать `DISTINCT` как пластырь против неверного `JOIN`.
- Фильтровать агрегаты в `WHERE`.
- Считать `AVG` от уже агрегированных средних без весов.

## Оптимизация запросов: база

Оптимизация начинается не с индекса, а с понимания:

- Какая бизнес-задача решается.
- Сколько строк в таблицах и какова селективность условий.
- Какие связи и ограничения есть: primary key, foreign key, unique.
- Какие запросы реально выполняются и с какими параметрами.
- Что показывает фактический план.

Частые причины медленных запросов:

- Полные сканы больших таблиц при низкой селективности фильтра.
- Неподходящие или отсутствующие индексы.
- Несаржабельные условия.
- Сортировки и группировки с большим объемом данных.
- Ошибочная кардинальность из-за устаревшей статистики.
- N+1 запросы на уровне приложения.
- Избыточные `JOIN`, `DISTINCT`, `ORDER BY`.
- Большие `OFFSET` для пагинации.

## Индексы и sargability

Sargable-условие позволяет СУБД эффективно использовать индекс. Обычно это условие вида `column operator constant`.

Хорошо:

```sql
SELECT *
FROM orders
WHERE created_at >= TIMESTAMP '2026-01-01'
  AND created_at <  TIMESTAMP '2026-02-01';
```

Плохо:

```sql
SELECT *
FROM orders
WHERE DATE(created_at) = DATE '2026-01-01';
```

Лучше переписать диапазоном или создать expression index, если это действительно нужно:

```sql
CREATE INDEX idx_orders_created_date ON orders ((DATE(created_at)));
```

Важные типы индексов PostgreSQL:

- B-tree - основной выбор для `=`, range, `ORDER BY`.
- Hash - редко нужен явно, B-tree обычно достаточно для `=`.
- GIN - массивы, `jsonb`, full-text search.
- GiST/SP-GiST - геометрия, ranges, специальные типы.
- BRIN - очень большие физически упорядоченные таблицы, например события по времени.

Композитный индекс:

```sql
CREATE INDEX idx_orders_customer_created ON orders (customer_id, created_at DESC);
```

Подходит для:

```sql
WHERE customer_id = 42
ORDER BY created_at DESC
```

Правило левого префикса полезно как модель: индекс `(a, b, c)` хорошо работает для условий по `a`, `a,b`, `a,b,c`; для одного `b` обычно хуже.

Partial index:

```sql
CREATE INDEX idx_orders_paid_created
ON orders (created_at DESC)
WHERE status = 'paid';
```

Covering index в PostgreSQL через `INCLUDE`:

```sql
CREATE INDEX idx_orders_customer_include
ON orders (customer_id)
INCLUDE (amount, status);
```

На интервью важно сказать: индекс ускоряет чтение не бесплатно. Он замедляет вставки/обновления, занимает место, требует обслуживания и может не использоваться при низкой селективности.

## EXPLAIN и EXPLAIN ANALYZE

`EXPLAIN` показывает план без выполнения. `EXPLAIN ANALYZE` выполняет запрос и показывает фактические времена и строки. Для изменяющих запросов используйте транзакцию с `ROLLBACK`, если нельзя менять данные.

```sql
BEGIN;
EXPLAIN (ANALYZE, BUFFERS, VERBOSE)
DELETE FROM sessions
WHERE expires_at < now();
ROLLBACK;
```

На что смотреть:

- `Seq Scan`, `Index Scan`, `Index Only Scan`, `Bitmap Heap Scan`.
- `Nested Loop`, `Hash Join`, `Merge Join`.
- `Sort`, `HashAggregate`, `GroupAggregate`.
- Estimated rows vs actual rows.
- `Buffers`: чтение из cache/disk.
- Время на узлах, loops, memory, spills to disk.

Пример объяснения:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT *
FROM orders
WHERE customer_id = 42
ORDER BY created_at DESC
LIMIT 20;
```

Если план делает `Seq Scan + Sort` по миллионам строк, вероятно нужен индекс `(customer_id, created_at DESC)`.

## Оконные функции

Оконная функция считает значение по набору строк, но не сворачивает строки как `GROUP BY`.

Базовый синтаксис:

```sql
function(...) OVER (
    PARTITION BY ...
    ORDER BY ...
    ROWS BETWEEN ... AND ...
)
```

### PARTITION BY и ORDER BY

```sql
SELECT
    id,
    customer_id,
    amount,
    created_at,
    ROW_NUMBER() OVER (
        PARTITION BY customer_id
        ORDER BY created_at DESC
    ) AS order_num
FROM orders;
```

Получить последний заказ каждого клиента:

```sql
SELECT *
FROM (
    SELECT
        o.*,
        ROW_NUMBER() OVER (
            PARTITION BY customer_id
            ORDER BY created_at DESC, id DESC
        ) AS rn
    FROM orders o
) t
WHERE rn = 1;
```

### Ranking

```sql
SELECT
    customer_id,
    SUM(amount) AS total,
    ROW_NUMBER() OVER (ORDER BY SUM(amount) DESC) AS row_number,
    RANK()       OVER (ORDER BY SUM(amount) DESC) AS rank,
    DENSE_RANK() OVER (ORDER BY SUM(amount) DESC) AS dense_rank
FROM orders
WHERE status = 'paid'
GROUP BY customer_id;
```

Разница:

- `ROW_NUMBER` всегда уникальный номер.
- `RANK` дает одинаковый ранг равным значениям и оставляет пропуски.
- `DENSE_RANK` дает одинаковый ранг равным значениям без пропусков.

### Running totals

```sql
SELECT
    id,
    customer_id,
    amount,
    created_at,
    SUM(amount) OVER (
        PARTITION BY customer_id
        ORDER BY created_at, id
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS running_total
FROM orders
WHERE status = 'paid';
```

### Frames: ROWS, RANGE, GROUPS

Frame определяет, какие строки внутри окна участвуют в расчете для текущей строки.

```sql
AVG(amount) OVER (
    ORDER BY created_at
    ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
) AS moving_avg_7_rows
```

Важная ловушка PostgreSQL: если в окне есть `ORDER BY`, frame по умолчанию часто ведет себя как `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`, то есть включает peers с одинаковым значением сортировки. Для running total часто безопаснее явно писать `ROWS` и добавлять tie-breaker `id`.

### LAG и LEAD

```sql
SELECT
    id,
    customer_id,
    amount,
    amount - LAG(amount) OVER (
        PARTITION BY customer_id
        ORDER BY created_at, id
    ) AS diff_from_previous
FROM orders;
```

ClickHouse поддерживает оконные функции, но синтаксис и производительность могут отличаться; для аналитических нагрузок важно смотреть документацию конкретной версии.

## CTE: WITH

CTE улучшает читаемость и позволяет переиспользовать промежуточный результат в запросе.

```sql
WITH paid_orders AS (
    SELECT *
    FROM orders
    WHERE status = 'paid'
), customer_totals AS (
    SELECT customer_id, SUM(amount) AS total
    FROM paid_orders
    GROUP BY customer_id
)
SELECT c.id, c.email, ct.total
FROM customers c
JOIN customer_totals ct ON ct.customer_id = c.id
WHERE ct.total > 1000;
```

### Материализация CTE

В PostgreSQL до 12 версии CTE часто был optimization fence: оптимизатор материализовал CTE и хуже проталкивал фильтры. В PostgreSQL 12+ CTE может inline-иться, а также есть явные подсказки:

```sql
WITH paid_orders AS NOT MATERIALIZED (
    SELECT * FROM orders WHERE status = 'paid'
)
SELECT COUNT(*) FROM paid_orders WHERE amount > 100;
```

```sql
WITH expensive AS MATERIALIZED (
    SELECT customer_id, SUM(amount) AS total
    FROM orders
    GROUP BY customer_id
)
SELECT * FROM expensive WHERE total > 1000;
```

На интервью: CTE не всегда быстрее подзапроса. Это инструмент читаемости и декомпозиции; производительность проверяется планом.

### Recursive CTE

Рекурсивные CTE полезны для деревьев, графов, иерархий.

```sql
WITH RECURSIVE category_tree AS (
    SELECT id, parent_id, name, 1 AS depth, ARRAY[id] AS path
    FROM categories
    WHERE id = 10

    UNION ALL

    SELECT c.id, c.parent_id, c.name, ct.depth + 1, ct.path || c.id
    FROM categories c
    JOIN category_tree ct ON c.parent_id = ct.id
    WHERE NOT c.id = ANY(ct.path)
)
SELECT *
FROM category_tree
ORDER BY path;
```

Важные детали:

- Базовая часть задает старт.
- Рекурсивная часть добавляет следующий уровень.
- Нужен контроль циклов.
- `UNION ALL` быстрее, но не убирает дубли.
- Глубокие деревья могут быть дорогими; иногда closure table/materialized path лучше.

## Подзапросы

Некоррелированный подзапрос не зависит от внешней строки и может быть выполнен один раз или оптимизирован.

```sql
SELECT *
FROM orders
WHERE amount > (SELECT AVG(amount) FROM orders);
```

Коррелированный подзапрос зависит от внешней строки.

```sql
SELECT c.*
FROM customers c
WHERE EXISTS (
    SELECT 1
    FROM orders o
    WHERE o.customer_id = c.id
      AND o.status = 'paid'
);
```

### EXISTS vs IN

`EXISTS` хорошо выражает проверку существования и безопасен с `NULL` в anti-join через `NOT EXISTS`.

```sql
SELECT c.*
FROM customers c
WHERE EXISTS (
    SELECT 1 FROM orders o WHERE o.customer_id = c.id
);
```

`IN` удобен для небольших списков или простых подзапросов:

```sql
SELECT *
FROM orders
WHERE status IN ('paid', 'refunded');
```

В современных PostgreSQL оптимизатор часто приводит `EXISTS` и `IN` к близким планам, но `NOT IN` с `NULL` остается частой ловушкой.

### Scalar subquery

Скалярный подзапрос должен вернуть одно значение. Если вернет несколько строк, будет ошибка.

```sql
SELECT
    c.id,
    c.email,
    (
        SELECT MAX(o.created_at)
        FROM orders o
        WHERE o.customer_id = c.id
    ) AS last_order_at
FROM customers c;
```

Иногда это читаемо, но для больших выборок может проиграть `JOIN` с предварительной агрегацией. Проверять `EXPLAIN ANALYZE`.

## Транзакции

Транзакция - атомарная единица работы с БД.

```sql
BEGIN;

UPDATE accounts
SET balance = balance - 100
WHERE id = 1;

UPDATE accounts
SET balance = balance + 100
WHERE id = 2;

COMMIT;
```

При ошибке:

```sql
ROLLBACK;
```

### ACID

- Atomicity - все операции транзакции применяются целиком или не применяются.
- Consistency - транзакция переводит данные из одного корректного состояния в другое.
- Isolation - параллельные транзакции не должны некорректно влиять друг на друга.
- Durability - после `COMMIT` данные переживают сбои в рамках гарантий СУБД.

### Уровни изоляции

Стандарт SQL описывает:

- `READ UNCOMMITTED` - могут быть dirty reads; PostgreSQL фактически ведет себя как `READ COMMITTED`.
- `READ COMMITTED` - каждая команда видит снимок данных на начало команды. Дефолт PostgreSQL.
- `REPEATABLE READ` - транзакция видит стабильный снимок на начало транзакции; в PostgreSQL предотвращает non-repeatable reads и phantom reads для обычных чтений благодаря MVCC snapshot.
- `SERIALIZABLE` - результат как будто транзакции выполнены последовательно; возможны serialization failures, приложение должно retry.

Пример:

```sql
BEGIN ISOLATION LEVEL SERIALIZABLE;
-- business logic
COMMIT;
```

Senior-level ответ: высокий уровень изоляции не означает отсутствие ошибок приложения. Нужно проектировать идемпотентность, ретраи, порядок блокировок, уникальные ограничения и проверять инварианты в БД.

### Locks

PostgreSQL использует разные блокировки: row-level locks, table locks, predicate locks при serializable, advisory locks.

```sql
SELECT *
FROM jobs
WHERE status = 'pending'
ORDER BY id
FOR UPDATE SKIP LOCKED
LIMIT 10;
```

`FOR UPDATE SKIP LOCKED` полезен для очередей воркеров, но требует аккуратной модели повторной обработки и мониторинга зависших задач.

### Deadlocks

Deadlock возникает, когда транзакции ждут друг друга по циклу.

Пример плохого порядка:

- Транзакция A заблокировала account 1, потом хочет account 2.
- Транзакция B заблокировала account 2, потом хочет account 1.

Профилактика:

- Блокировать ресурсы в стабильном порядке.
- Держать транзакции короткими.
- Не выполнять внешние HTTP/API вызовы внутри транзакции.
- Индексировать условия обновлений и удалений.
- Обрабатывать deadlock/serialization errors retry-логикой.

### MVCC

MVCC позволяет читателям и писателям меньше блокировать друг друга. PostgreSQL хранит версии строк, а транзакции читают подходящую версию согласно snapshot.

Практические последствия:

- Долгие транзакции мешают очистке старых версий.
- Нужны `VACUUM` и актуальная статистика.
- `UPDATE` фактически создает новую версию строки.
- Частые обновления индексируемых колонок дороже.

## PHP backend: практические замечания

- N+1 часто дороже, чем один правильно написанный SQL.
- Не склеивайте SQL строками с пользовательским вводом; используйте prepared statements/bind parameters.
- Для денежных значений используйте `numeric`/`decimal` или целые minor units, не `float`.
- Пагинация через большой `OFFSET` деградирует; для лент лучше keyset pagination.
- Бизнес-инварианты закрепляйте ограничениями БД: `UNIQUE`, `CHECK`, `FOREIGN KEY`, `EXCLUDE`, partial unique index.
- В Laravel/Doctrine внимательно проверяйте генерируемый SQL и количество запросов.
- Транзакции должны быть короткими; не держите их во время сетевых вызовов, рендера, очередей, файловых операций.

Пример keyset pagination:

```sql
SELECT *
FROM orders
WHERE (created_at, id) < (:last_created_at, :last_id)
ORDER BY created_at DESC, id DESC
LIMIT 50;
```

Индекс:

```sql
CREATE INDEX idx_orders_created_id_desc ON orders (created_at DESC, id DESC);
```

## Частые ловушки

- `LEFT JOIN` ломается условием правой таблицы в `WHERE`.
- `COUNT(column)` не считает `NULL`.
- `NOT IN` с `NULL` дает неожиданный результат.
- `DISTINCT` скрывает ошибку кардинальности.
- `LIMIT` без стабильного `ORDER BY` дает недетерминированный результат.
- `ORDER BY created_at` без tie-breaker может давать нестабильную пагинацию.
- Индекс на колонку не помогает, если колонка обернута функцией без expression index.
- Индекс `(a, b)` не всегда полезен для фильтра только по `b`.
- CTE не является гарантированной оптимизацией.
- Оконные функции не фильтруются в `WHERE` того же уровня.
- Долгие транзакции портят производительность MVCC.
- `SELECT *` в API и фоновых задачах часто тащит лишние данные и ломает покрывающие индексы.

## Интервью: короткие senior-level ответы

**Чем `WHERE` отличается от `HAVING`?**

`WHERE` фильтрует строки до группировки, `HAVING` фильтрует группы после агрегирования. Если условие не зависит от агрегата, лучше держать его в `WHERE`, чтобы уменьшить объем данных раньше.

**Когда использовать `EXISTS`, а когда `JOIN`?**

Если нужно проверить существование связанной строки, `EXISTS` обычно точнее выражает намерение и не размножает строки. Если нужны данные из обеих таблиц, нужен `JOIN`.

**Почему запрос с индексом все равно делает Seq Scan?**

Потому что оптимизатор считает Seq Scan дешевле: условие низкоселективное, таблица маленькая, статистика говорит о большом числе строк, условие несаржабельное, типы не совпадают или индекс не подходит под фильтр/сортировку.

**Что такое MVCC?**

Механизм версионности строк: транзакции читают согласованный снимок данных, а писатели создают новые версии. Это уменьшает блокировки чтения, но требует очистки старых версий и осторожности с долгими транзакциями.

**Что делать с deadlock?**

Считать его нормальной ошибкой конкурентной системы: логировать, retry-ить безопасные операции, уменьшать длительность транзакций, блокировать ресурсы в одинаковом порядке, добавлять индексы на условия блокирующих `UPDATE`/`DELETE`.

**`ROW_NUMBER`, `RANK`, `DENSE_RANK`: разница?**

`ROW_NUMBER` всегда уникален. `RANK` дает одинаковый ранг равным строкам и пропускает следующие номера. `DENSE_RANK` дает одинаковый ранг без пропусков.

**CTE быстрее подзапроса?**

Не обязательно. CTE часто нужен для читаемости. В PostgreSQL 12+ он может быть inline-нут, но `MATERIALIZED`/`NOT MATERIALIZED` и фактический план важнее догадок.

**Как оптимизировать slow query?**

Сначала собрать `EXPLAIN (ANALYZE, BUFFERS)`, реальные параметры, объем данных и частоту запроса. Затем проверить кардинальность, фильтры, индексы, сортировки, join order, N+1, возможность переписать запрос или изменить модель данных.

## Mini-practice задачи

1. Найти клиентов без заказов. Написать через `LEFT JOIN ... IS NULL` и через `NOT EXISTS`. Объяснить, какой вариант безопаснее с `NULL`.
2. Для каждого клиента вывести последний paid-заказ. Решить через `ROW_NUMBER` и через `DISTINCT ON` в PostgreSQL.
3. Посчитать дневную выручку и накопительную выручку по дням.
4. Найти топ-3 клиента по выручке в каждом регионе.
5. Переписать запрос `WHERE DATE(created_at) = CURRENT_DATE` в sargable-форму.
6. По плану `Seq Scan + Sort` предложить индекс для запроса с `WHERE customer_id = ? ORDER BY created_at DESC LIMIT 20`.
7. Смоделировать deadlock с двумя таблицами/строками и предложить стабильный порядок блокировок.
8. Написать recursive CTE для дерева категорий с защитой от циклов.
9. Сравнить `NOT IN`, `NOT EXISTS`, `LEFT JOIN ... IS NULL` на данных, где в дочерней таблице есть `NULL`.
10. Переписать N+1 получение счетчиков заказов клиентов в один агрегирующий запрос.

## Вопросы для самопроверки

1. Почему `LEFT JOIN` может неожиданно стать `INNER JOIN`?
2. Почему `COUNT(*)` и `COUNT(column)` дают разные результаты?
3. Что означает estimated rows vs actual rows в `EXPLAIN ANALYZE`?
4. Когда композитный индекс `(a, b, c)` полезен, а когда нет?
5. Чем `ROWS` отличается от `RANGE` в оконных frame?
6. Почему `LIMIT/OFFSET` плохо масштабируется?
7. Какие аномалии возможны при `READ COMMITTED`?
8. Почему долгие транзакции вредны в PostgreSQL?
9. Когда `FOR UPDATE SKIP LOCKED` уместен?
10. Почему `DISTINCT` в конце запроса может быть запахом проблемы?

## Code review checklist для SQL

- Запрос выражает нужную кардинальность: `JOIN` не размножает строки случайно.
- Условия для `LEFT JOIN` стоят в `ON`, если нужно сохранить строки слева.
- Нет опасного `NOT IN` с потенциальными `NULL`.
- Нет `SELECT *` там, где нужен ограниченный набор полей.
- Есть стабильный `ORDER BY` для пагинации и оконных функций.
- Фильтры sargable; колонки не обернуты функциями без expression index.
- Индексы соответствуют фильтрам, `JOIN`, сортировке и лимитам.
- `EXPLAIN (ANALYZE, BUFFERS)` проверен для критичных запросов.
- Агрегация выполняется на правильном уровне детализации.
- `HAVING` не используется вместо обычного `WHERE` без необходимости.
- CTE не скрывает проблему оптимизации; материализация осознана.
- Транзакции короткие, без внешних сетевых вызовов.
- Конкурентные операции имеют retry для deadlock/serialization failure.
- Инварианты защищены ограничениями БД, а не только PHP-кодом.

## Ссылки

- PostgreSQL Documentation: Queries - https://www.postgresql.org/docs/current/queries.html
- PostgreSQL Documentation: Table Expressions / Joins - https://www.postgresql.org/docs/current/queries-table-expressions.html
- PostgreSQL Documentation: Aggregate Functions - https://www.postgresql.org/docs/current/functions-aggregate.html
- PostgreSQL Documentation: Window Functions - https://www.postgresql.org/docs/current/functions-window.html
- PostgreSQL Documentation: Window Function Calls - https://www.postgresql.org/docs/current/sql-expressions.html#SYNTAX-WINDOW-FUNCTIONS
- PostgreSQL Documentation: WITH Queries / Recursive Queries - https://www.postgresql.org/docs/current/queries-with.html
- PostgreSQL Documentation: EXPLAIN - https://www.postgresql.org/docs/current/sql-explain.html
- PostgreSQL Documentation: Using EXPLAIN - https://www.postgresql.org/docs/current/using-explain.html
- PostgreSQL Documentation: Indexes - https://www.postgresql.org/docs/current/indexes.html
- PostgreSQL Documentation: Transaction Isolation - https://www.postgresql.org/docs/current/transaction-iso.html
- PostgreSQL Documentation: Explicit Locking - https://www.postgresql.org/docs/current/explicit-locking.html
- PostgreSQL Documentation: MVCC - https://www.postgresql.org/docs/current/mvcc.html
- MySQL Documentation: JOIN Clause - https://dev.mysql.com/doc/refman/8.4/en/join.html
- MySQL Documentation: Window Functions - https://dev.mysql.com/doc/refman/8.4/en/window-functions.html
- MySQL Documentation: InnoDB Transaction Model and Locking - https://dev.mysql.com/doc/refman/8.4/en/innodb-transaction-model.html
- ClickHouse Documentation: SELECT - https://clickhouse.com/docs/sql-reference/statements/select
- ClickHouse Documentation: Window Functions - https://clickhouse.com/docs/sql-reference/window-functions
- Use The Index, Luke - https://use-the-index-luke.com/
- Markus Winand: SQL Performance Explained - https://sql-performance-explained.com/
- Markus Winand: Modern SQL - https://modern-sql.com/
- PostgreSQL EXPLAIN Visualizer by Dalibo - https://explain.dalibo.com/
