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

## Производительность запросов

Этот файл фокусируется на SQL-семантике: кардинальность, фильтрация, группировка,
оконные функции, CTE и подзапросы.

Индексы, sargability, `EXPLAIN`, статистика planner-а, `Seq Scan`, `Bitmap Scan`,
`Index Only Scan` и диагностика slow query разобраны в canonical-файле
[`02 PostgreSQL production internals.md`](02%20PostgreSQL%20production%20internals.md).

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

## Транзакции и конкурентность

Транзакции, ACID, уровни изоляции, MVCC, locks, deadlocks, `FOR UPDATE SKIP LOCKED`,
serialization failures и retry-логика разобраны в canonical-файле
[`02 PostgreSQL production internals.md`](02%20PostgreSQL%20production%20internals.md).

В этом SQL-конспекте остаются темы языка запросов. Если вопрос интервью уходит в
конкурентность или production-поведение PostgreSQL, переходи в `02`.

## PHP backend: практические замечания

- Не склеивайте SQL строками с пользовательским вводом; используйте prepared statements/bind parameters.
- Для денежных значений используйте `numeric`/`decimal` или целые minor units, не `float`.
- Для лент лучше keyset pagination, чем большой `OFFSET`; production-детали см. в `02`.
- В Laravel/Doctrine внимательно проверяйте генерируемый SQL и количество запросов.

Пример keyset pagination:

```sql
SELECT *
FROM orders
WHERE (created_at, id) < (:last_created_at, :last_id)
ORDER BY created_at DESC, id DESC
LIMIT 50;
```

## Частые ловушки

- `LEFT JOIN` ломается условием правой таблицы в `WHERE`.
- `COUNT(column)` не считает `NULL`.
- `NOT IN` с `NULL` дает неожиданный результат.
- `DISTINCT` скрывает ошибку кардинальности.
- `LIMIT` без стабильного `ORDER BY` дает недетерминированный результат.
- `ORDER BY created_at` без tie-breaker может давать нестабильную пагинацию.
- CTE не является гарантированной оптимизацией.
- Оконные функции не фильтруются в `WHERE` того же уровня.
- `SELECT *` в API и фоновых задачах часто тащит лишние данные и ломает покрывающие индексы.

## Интервью: короткие senior-level ответы

**Чем `WHERE` отличается от `HAVING`?**

`WHERE` фильтрует строки до группировки, `HAVING` фильтрует группы после агрегирования. Если условие не зависит от агрегата, лучше держать его в `WHERE`, чтобы уменьшить объем данных раньше.

**Когда использовать `EXISTS`, а когда `JOIN`?**

Если нужно проверить существование связанной строки, `EXISTS` обычно точнее выражает намерение и не размножает строки. Если нужны данные из обеих таблиц, нужен `JOIN`.

**`ROW_NUMBER`, `RANK`, `DENSE_RANK`: разница?**

`ROW_NUMBER` всегда уникален. `RANK` дает одинаковый ранг равным строкам и пропускает следующие номера. `DENSE_RANK` дает одинаковый ранг без пропусков.

**CTE быстрее подзапроса?**

Не обязательно. CTE часто нужен для читаемости. В PostgreSQL 12+ он может быть inline-нут, но `MATERIALIZED`/`NOT MATERIALIZED` и фактический план важнее догадок.

## Mini-practice задачи

1. Найти клиентов без заказов. Написать через `LEFT JOIN ... IS NULL` и через `NOT EXISTS`. Объяснить, какой вариант безопаснее с `NULL`.
2. Для каждого клиента вывести последний paid-заказ. Решить через `ROW_NUMBER` и через `DISTINCT ON` в PostgreSQL.
3. Посчитать дневную выручку и накопительную выручку по дням.
4. Найти топ-3 клиента по выручке в каждом регионе.
5. Написать recursive CTE для дерева категорий с защитой от циклов.
6. Сравнить `NOT IN`, `NOT EXISTS`, `LEFT JOIN ... IS NULL` на данных, где в дочерней таблице есть `NULL`.
7. Переписать N+1 получение счетчиков заказов клиентов в один агрегирующий запрос.

Практика по индексам, `EXPLAIN`, deadlocks и slow query находится в
[`02 PostgreSQL production internals.md`](02%20PostgreSQL%20production%20internals.md).

## Вопросы для самопроверки

1. Почему `LEFT JOIN` может неожиданно стать `INNER JOIN`?
2. Почему `COUNT(*)` и `COUNT(column)` дают разные результаты?
3. Чем `ROWS` отличается от `RANGE` в оконных frame?
4. Почему `LIMIT/OFFSET` плохо масштабируется?
5. Почему `DISTINCT` в конце запроса может быть запахом проблемы?

## Code review checklist для SQL

- Запрос выражает нужную кардинальность: `JOIN` не размножает строки случайно.
- Условия для `LEFT JOIN` стоят в `ON`, если нужно сохранить строки слева.
- Нет опасного `NOT IN` с потенциальными `NULL`.
- Нет `SELECT *` там, где нужен ограниченный набор полей.
- Есть стабильный `ORDER BY` для пагинации и оконных функций.
- Агрегация выполняется на правильном уровне детализации.
- `HAVING` не используется вместо обычного `WHERE` без необходимости.
- CTE не скрывает проблему оптимизации; материализация осознана.
- Production-проверки по индексам, планам, locks и транзакциям выполняются по checklist из `02`.

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
