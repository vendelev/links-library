# API Design, REST, OpenAPI, идемпотентность, версионирование

Гайд для подготовки Lead/Senior PHP backend developer к интервью. Фокус: практические решения, компромиссы, типовые ошибки и короткие senior-level ответы.

## Быстрая карта темы

- API design: контракт между клиентом и сервером, а не просто набор роутов.
- REST: архитектурный стиль с ресурсами, стандартной семантикой HTTP и stateless-взаимодействием.
- OpenAPI: машинно-читаемый контракт для документации, генерации клиентов, моков и contract testing.
- Идемпотентность: повтор запроса не должен создавать неожиданные побочные эффекты.
- Версионирование: управление изменениями контракта без поломки клиентов.

## REST: ограничения и смысл

REST задает не URL-стиль, а набор архитектурных ограничений.

### Основные constraints

- Client-server: клиент и сервер разделены, UI не знает внутреннюю модель БД.
- Stateless: каждый запрос содержит всю информацию для обработки; сервер не хранит состояние сессии запроса.
- Cacheable: ответы явно описывают, можно ли их кешировать.
- Uniform interface: единый интерфейс через ресурсы, HTTP-методы, media types, ссылки.
- Layered system: клиент не обязан знать, общается ли он с origin server, gateway, CDN или proxy.
- Code on demand: необязательное ограничение, редко важно для backend API.

Senior-level ответ:

> REST важен не сам по себе, а как способ сделать API предсказуемым: ресурсы вместо RPC-действий, стандартная семантика HTTP, stateless-запросы, кеширование и независимая эволюция клиента и сервера.

### Частые ошибки

- Называть REST любой JSON-over-HTTP API.
- Использовать `POST /getUser` вместо `GET /users/{id}`.
- Игнорировать HTTP-коды и всегда возвращать `200`.
- Хранить серверное состояние между запросами без явной модели.
- Проектировать API из структуры таблиц, а не из доменной модели и сценариев клиента.

## Ресурсы и URL

Ресурс - доменная сущность или коллекция, доступная по стабильному идентификатору.

Примеры:

```http
GET /users
GET /users/{userId}
GET /users/{userId}/orders
POST /orders
PATCH /orders/{orderId}
```

Рекомендации:

- Использовать существительные во множественном числе: `/users`, `/orders`.
- Не раскрывать внутренние ID, если это риск: можно использовать UUID/ULID/public ID.
- Не делать глубокую вложенность без необходимости: `/users/{id}/orders/{orderId}/items/{itemId}` быстро становится хрупким.
- Подресурс оправдан, если он существует только в контексте родителя или часто выбирается в этом контексте.
- Действия моделировать как ресурсы, если они имеют состояние: `POST /payments/{id}/captures`, `POST /orders/{id}/cancellations`.

Компромисс:

- Чистый REST удобен для CRUD и доменных ресурсов.
- Для сложных команд иногда практичнее command endpoint: `POST /orders/{id}:cancel` или `POST /order-cancellations`. Главное - явно описать контракт, идемпотентность и ошибки.

## HTTP-методы

### Семантика

| Метод | Назначение | Safe | Idempotent | Типичный body |
| --- | --- | --- | --- | --- |
| `GET` | Получить ресурс | Да | Да | Нет |
| `HEAD` | Метаданные без body | Да | Да | Нет |
| `OPTIONS` | Доступные методы/CORS | Да | Да | Нет |
| `POST` | Создать ресурс или выполнить команду | Нет | Нет по умолчанию | Да |
| `PUT` | Полная замена ресурса по известному URI | Нет | Да | Да |
| `PATCH` | Частичное изменение ресурса | Нет | Не всегда | Да |
| `DELETE` | Удалить ресурс | Нет | Да по семантике | Обычно нет |

Важно:

- Idempotent не значит "без побочных эффектов". `DELETE /users/1` может удалить пользователя, но повторный `DELETE` не должен удалять что-то еще.
- `POST` можно сделать идемпотентным через `Idempotency-Key`.
- `PATCH` может быть идемпотентным, если формат операции идемпотентен, например `{"status":"cancelled"}`. Неидемпотентный пример: `{"increment":1}`.

### PUT vs PATCH

`PUT`:

- Клиент отправляет полное представление ресурса.
- Повтор запроса приводит к тому же состоянию.
- Опасность: случайно затереть поля, неизвестные клиенту.

`PATCH`:

- Клиент отправляет изменения.
- Удобен для частичного обновления.
- Нужно выбрать формат: JSON Merge Patch (`application/merge-patch+json`) или JSON Patch (`application/json-patch+json`).

Senior-level ответ:

> Я использую `PUT`, когда клиент владеет полным представлением ресурса, и `PATCH`, когда обновляет ограниченный набор полей. Для конкурентных обновлений добавляю `If-Match` с ETag или version field.

## HTTP-статусы

### Успешные ответы

| Код | Когда использовать |
| --- | --- |
| `200 OK` | Успешный запрос с body |
| `201 Created` | Ресурс создан, желательно `Location` header |
| `202 Accepted` | Запрос принят на асинхронную обработку |
| `204 No Content` | Успех без body, часто `DELETE` или `PATCH` |

### Ошибки клиента

| Код | Когда использовать |
| --- | --- |
| `400 Bad Request` | Некорректный синтаксис или общий плохой запрос |
| `401 Unauthorized` | Нет или неверная аутентификация |
| `403 Forbidden` | Аутентифицирован, но нет прав |
| `404 Not Found` | Ресурс не найден или скрыт политикой доступа |
| `405 Method Not Allowed` | Метод не поддержан для ресурса |
| `409 Conflict` | Конфликт состояния, например duplicate или business conflict |
| `412 Precondition Failed` | Не прошел `If-Match`/`If-Unmodified-Since` |
| `415 Unsupported Media Type` | Неподдерживаемый `Content-Type` |
| `422 Unprocessable Content` | Валидный JSON, но ошибки валидации доменных полей |
| `429 Too Many Requests` | Rate limit превышен |

### Ошибки сервера

| Код | Когда использовать |
| --- | --- |
| `500 Internal Server Error` | Неожиданная ошибка |
| `502 Bad Gateway` | Ошибка upstream-сервиса через gateway |
| `503 Service Unavailable` | Сервис временно недоступен |
| `504 Gateway Timeout` | Timeout upstream-сервиса |

Частые pitfalls:

- `401` vs `403`: `401` про аутентификацию, `403` про авторизацию.
- `400` vs `422`: `400` для плохого формата, `422` для семантической валидации.
- `409` vs `422`: `409` для конфликта текущего состояния системы, `422` для неправильных входных данных.
- Не возвращать stack trace, SQL errors, class names во внешнее API.

## Request/response design

### Общие принципы

- Стабильный контракт важнее удобства текущей реализации.
- Имена полей должны быть единообразны: `snake_case` или `camelCase`, но не смесь.
- Даты передавать в ISO 8601/RFC 3339: `2026-06-29T12:30:00Z`.
- Деньги передавать как minor units (`amount_cents`) или decimal string плюс currency, не float.
- Enum значения документировать и расширять осторожно.
- Nullable и absent fields различать явно.
- Не отдавать внутренние поля: `password_hash`, internal status, foreign keys без смысла для клиента.

Пример ответа:

```json
{
  "id": "ord_01J2...",
  "status": "paid",
  "amount": {
    "value": "1299.00",
    "currency": "EUR"
  },
  "created_at": "2026-06-29T12:30:00Z"
}
```

### Envelope или bare resource

Bare resource:

```json
{
  "id": "usr_123",
  "email": "user@example.com"
}
```

Envelope:

```json
{
  "data": {
    "id": "usr_123",
    "email": "user@example.com"
  },
  "meta": {
    "request_id": "req_123"
  }
}
```

Tradeoff:

- Bare resource проще и ближе к HTTP.
- Envelope удобен для `meta`, pagination, tracing, uniform error handling.
- Главное - единообразие.

## Формат ошибок

Практичный вариант - RFC 9457 Problem Details (`application/problem+json`).

```json
{
  "type": "https://api.example.com/problems/validation-error",
  "title": "Validation failed",
  "status": 422,
  "detail": "One or more fields are invalid.",
  "instance": "/orders",
  "request_id": "req_123",
  "errors": [
    {
      "field": "email",
      "code": "invalid_email",
      "message": "Email must be valid."
    }
  ]
}
```

Рекомендации:

- `code` должен быть стабильным для программной обработки.
- `message` можно локализовать, но не строить на нем логику клиента.
- Добавлять `request_id`/`trace_id` для поддержки и observability.
- Не раскрывать внутренние детали.

Senior-level ответ:

> Для ошибок я предпочитаю Problem Details с расширением `errors` для field-level validation. Внешний контракт получает стабильные error codes, а не тексты исключений.

## Валидация

Уровни валидации:

- Transport/schema validation: JSON валиден, типы полей корректны, required fields присутствуют.
- Application validation: ограничения use case, например `start_date <= end_date`.
- Domain validation: бизнес-инварианты, например заказ нельзя оплатить дважды.
- Persistence constraints: unique indexes, foreign keys, check constraints как последняя линия защиты.

Практика:

- Не полагаться только на DTO validation; гонки все равно ловятся БД.
- Unique conflict лучше маппить в `409 Conflict` или `422`, в зависимости от контекста.
- Сообщения валидации должны быть безопасны для клиента.
- В OpenAPI документировать ограничения: `minLength`, `maxLength`, `format`, `enum`, `pattern`.

## Pagination, filtering, sorting

### Offset pagination

```http
GET /orders?limit=50&offset=100
```

Плюсы:

- Просто реализовать.
- Удобно для админок и страниц.

Минусы:

- Медленно на больших offset.
- Нестабильно при вставках/удалениях между запросами.

### Cursor pagination

```http
GET /orders?limit=50&cursor=eyJjcmVhdGVkX2F0Ijoi..."
```

Плюсы:

- Лучше для больших списков.
- Стабильнее при изменениях данных.

Минусы:

- Сложнее UX для произвольного перехода на страницу.
- Требует стабильной сортировки, например `created_at,id`.

### Filtering/sorting

```http
GET /orders?status=paid&created_from=2026-01-01&sort=-created_at,id
```

Рекомендации:

- Белый список полей фильтрации и сортировки.
- Ограничивать `limit` максимальным значением.
- Индексировать популярные фильтры.
- Документировать порядок сортировки и tie-breaker.
- Не давать клиенту произвольный SQL-like query без sandbox и лимитов.

## Идемпотентность

Идемпотентность означает, что повтор одного и того же запроса дает тот же итоговый эффект. Это критично для платежей, создания заказов, внешних интеграций и retries.

### Idempotency-Key

```http
POST /payments
Idempotency-Key: 01J2M7C4Z7K9...
Content-Type: application/json

{
  "order_id": "ord_123",
  "amount": "1299.00",
  "currency": "EUR"
}
```

Как работает:

- Клиент генерирует уникальный ключ на операцию.
- Сервер сохраняет key, request fingerprint, status и response.
- При повторе с тем же key и тем же payload возвращается тот же результат.
- При том же key, но другом payload - ошибка `409 Conflict` или `422`.
- Ключи имеют TTL, например 24 часа или больше для платежей.

Что хранить:

- `idempotency_key`.
- `request_hash`.
- `status`: processing/succeeded/failed.
- `response_status`, `response_body`.
- `created_at`, `expires_at`.

Pitfalls:

- Сохранять ключ после выполнения операции, а не до нее. При падении между side effect и записью ключа будет дубль.
- Не защищать запись ключа unique constraint.
- Не учитывать параллельные одинаковые запросы.
- Кешировать transient `500` навсегда.
- Использовать один ключ для разных операций.

Senior-level ответ:

> Для `POST` с побочными эффектами я требую `Idempotency-Key`, сохраняю его атомарно с request hash и итоговым response, защищаю unique index и корректно обрабатываю состояние `processing`, чтобы параллельные повторы не создали дубликаты.

## Retries и timeouts

Клиент не всегда знает, дошел ли запрос до сервера. Поэтому retry policy должен быть частью API-дизайна.

Рекомендации:

- Retry безопасен для `GET`, `HEAD`, `PUT`, `DELETE` по семантике, но все равно учитывать backend side effects.
- Для `POST` retry делать только с `Idempotency-Key`.
- Использовать exponential backoff + jitter.
- Уважать `Retry-After` для `429` и `503`.
- Различать connect timeout, read timeout, application timeout.
- Не ретраить бездумно `400`, `401`, `403`, `404`, `422`.
- Осторожно ретраить `409`, если конфликт временный.

## Optimistic locking, ETag, conditional requests

Проблема lost update:

1. Клиент A читает заказ версии 5.
2. Клиент B читает заказ версии 5.
3. B обновляет заказ до версии 6.
4. A отправляет старое обновление и затирает изменения B.

Решение через ETag:

```http
GET /orders/ord_123

HTTP/1.1 200 OK
ETag: "v5"
```

```http
PATCH /orders/ord_123
If-Match: "v5"
Content-Type: application/json

{
  "status": "cancelled"
}
```

Если версия уже изменилась:

```http
HTTP/1.1 412 Precondition Failed
```

Практика:

- ETag может быть version column, hash representation или updated_at с осторожностью.
- Для write operations лучше `If-Match`.
- Для кеширования `GET` можно использовать `If-None-Match` и `304 Not Modified`.
- Если API не использует ETag, можно передавать `version` в body, но это менее HTTP-native.

## API versioning

Версионирование нужно для breaking changes. Не каждое изменение требует новой версии.

### Стратегии

URI versioning:

```http
GET /v1/orders
```

Плюсы: просто, очевидно, удобно для routing и документации.

Минусы: версия становится частью ресурса, сложнее эволюция отдельных capabilities.

Header versioning:

```http
GET /orders
API-Version: 2026-06-29
```

Плюсы: URL чище, удобно для date-based versions.

Минусы: хуже discoverability, сложнее тестировать вручную.

Media type versioning:

```http
Accept: application/vnd.example.v1+json
```

Плюсы: близко к HTTP content negotiation.

Минусы: сложнее для клиентов и инфраструктуры.

Date-based versioning:

```http
Stripe-Version: 2024-06-20
```

Плюсы: удобно для SaaS API, можно закрепить поведение на аккаунт.

Минусы: сложная матрица поддержки.

Senior-level ответ:

> Для публичного API чаще выбираю `/v1` или date-based header, потому что это проще для клиентов и поддержки. Внутри версии стараюсь делать additive changes и не плодить версии без breaking change.

### Breaking vs non-breaking changes

Обычно non-breaking:

- Добавить новое необязательное поле в response.
- Добавить новый endpoint.
- Добавить необязательный request parameter.
- Расширить enum, если клиенты готовы к unknown values.
- Увеличить лимиты без изменения семантики.

Обычно breaking:

- Удалить или переименовать поле.
- Изменить тип поля.
- Сделать optional поле required.
- Изменить смысл status code или error code.
- Убрать enum value.
- Изменить pagination contract.
- Изменить authentication scheme.

Pitfall:

- Добавление enum value может быть breaking для клиентов с exhaustive switch. В публичном API надо документировать, что клиенты должны быть tolerant к unknown values.

## Backward compatibility

Правила совместимости:

- Prefer additive changes.
- Старые поля не удалять сразу, сначала deprecate.
- Не менять семантику существующего поля.
- Не переиспользовать старые error codes с новым смыслом.
- Клиенты должны игнорировать неизвестные response fields.
- Сервер должен быть строгим к request и толерантным к response evolution.
- Contract tests и consumer-driven contracts помогают ловить regressions.

Практический подход:

- Ввести новое поле рядом со старым.
- Заполнять оба поля в переходный период.
- Добавить метрики использования старого поля/endpoint.
- Предупредить клиентов.
- Удалить после официального sunset.

## Deprecation policy

Хорошая политика deprecation включает:

- Что deprecated: endpoint, поле, enum value, auth scheme.
- Когда deprecated: дата объявления.
- Когда sunset: дата отключения.
- Что использовать вместо этого.
- Каналы уведомления: changelog, email, dashboard, headers.
- Метрики использования и targeted communication.

Полезные headers:

```http
Deprecation: true
Sunset: Wed, 31 Dec 2026 23:59:59 GMT
Link: <https://docs.example.com/changelog/deprecations/orders-v1>; rel="deprecation"
```

Senior-level ответ:

> Я не удаляю контракт без политики deprecation: объявление, replacement, sunset date, мониторинг реального использования и коммуникация с владельцами клиентов.

## OpenAPI / Swagger

OpenAPI - спецификация HTTP API: endpoints, schemas, auth, параметры, ответы, ошибки, examples.

### Что документировать

- Paths и методы.
- Request parameters: path/query/header/cookie.
- Request body schemas.
- Response schemas по status codes.
- Error format.
- Auth/security schemes.
- Pagination и filtering.
- Rate limits и idempotency headers.
- Examples для happy path и ошибок.
- Deprecation markers.

Мини-пример:

```yaml
paths:
  /orders:
    post:
      summary: Create order
      parameters:
        - name: Idempotency-Key
          in: header
          required: true
          schema:
            type: string
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateOrderRequest'
      responses:
        '201':
          description: Created
          headers:
            Location:
              schema:
                type: string
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Order'
        '422':
          description: Validation error
          content:
            application/problem+json:
              schema:
                $ref: '#/components/schemas/Problem'
```

Практика:

- Contract-first: сначала OpenAPI, потом реализация. Хорошо для публичных API и нескольких команд.
- Code-first: генерировать OpenAPI из атрибутов/аннотаций. Удобно быстрее, но легко получить документацию реализации, а не контракта.
- Проверять OpenAPI в CI: lint, breaking changes, schema validation.
- Использовать examples как часть документации и тестов.

Pitfalls:

- Документация не соответствует реальности.
- Описан только `200`, нет ошибок.
- Нет примеров.
- Все поля `string`, constraints не заданы.
- OpenAPI генерируется, но не проходит review как контракт.

## Webhooks

Webhook - обратный вызов от сервера к клиенту при событии.

Пример:

```http
POST https://client.example.com/webhooks/payments
X-Webhook-Id: evt_123
X-Webhook-Timestamp: 1782739200
X-Webhook-Signature: v1=...

{
  "id": "evt_123",
  "type": "payment.succeeded",
  "created_at": "2026-06-29T12:30:00Z",
  "data": {
    "payment_id": "pay_123"
  }
}
```

Правила:

- Подписывать payload HMAC или asymmetric signature.
- Передавать timestamp и защищаться от replay attacks.
- Делать delivery at-least-once, поэтому получатель должен быть идемпотентным.
- Ретраить с backoff.
- Давать dashboard/log delivery attempts.
- Не требовать мгновенного ответа; клиент должен быстро вернуть `2xx` и обработать async.
- Версионировать event schema.

Pitfalls:

- Считать webhook exactly-once.
- Не проверять подпись.
- Отправлять PII без необходимости.
- Не иметь endpoint для ручного replay.

## Rate limiting

Цели:

- Защита от abuse.
- Fair usage между клиентами.
- Предсказуемость нагрузки.

Алгоритмы:

- Fixed window: просто, но есть burst на границе окна.
- Sliding window: точнее, дороже.
- Token bucket: позволяет burst в пределах bucket.
- Leaky bucket: сглаживает поток.

Headers:

```http
RateLimit-Limit: 1000
RateLimit-Remaining: 42
RateLimit-Reset: 60
Retry-After: 60
```

Практика:

- Лимитировать по API key/user/IP/tenant в зависимости от продукта.
- Разделять read/write limits.
- Для дорогих endpoints вводить отдельные quotas.
- Возвращать `429 Too Many Requests`.
- Документировать retry behavior.

## Authentication и authorization

Authentication отвечает на вопрос "кто ты?", authorization - "что тебе можно?".

Типовые схемы:

- Session cookie: удобно для browser apps, нужен CSRF protection.
- Bearer token/JWT: удобно для API, важно ограничивать TTL и scopes.
- OAuth 2.0/OIDC: стандарт для delegated access и SSO.
- API keys: просты для server-to-server, требуют rotation, scopes, audit.
- mTLS: сильная machine-to-machine аутентификация.

Рекомендации:

- Всегда HTTPS.
- Не передавать токены в query string.
- Использовать scopes/permissions.
- Логировать `subject`, `client_id`, `scope`, но не секреты.
- Делать key rotation и revocation.
- Для multi-tenant API проверять tenant boundary на каждом запросе.

Pitfalls:

- Проверять только наличие JWT, но не `aud`, `iss`, `exp`, signature.
- Полагаться на client-side authorization.
- Смешивать user permissions и service permissions.
- Возвращать `403` с подробностями, раскрывающими существование чужого ресурса.

## SOAP и legacy integrations

SOAP все еще встречается в enterprise, банках, госинтеграциях.

Что важно знать:

- Контракт описывается WSDL.
- Сообщения XML, часто строгие XSD schemas.
- Ошибки через SOAP Fault.
- WS-Security может включать подписи, encryption, timestamps.
- Может требоваться MTOM для binary attachments.

Интеграционный подход:

- Изолировать SOAP client за adapter/anti-corruption layer.
- Маппить legacy DTO в свою доменную модель.
- Делать retries только с пониманием идемпотентности операции.
- Логировать correlation IDs, но маскировать PII/secrets.
- Закладывать timeouts, circuit breaker, bulkhead.
- Покрывать контракт fixture-based tests.

Senior-level ответ:

> С legacy SOAP я не размазываю WSDL-generated классы по домену, а закрываю их adapter layer. Там же решаю retries, mapping, fault handling, timeout и observability.

## Laravel API notes

Полезные элементы Laravel:

- Routes в `routes/api.php`, route model binding.
- Form Request для validation и authorization на уровне use case.
- API Resources для response transformation.
- Policies/Gates для authorization.
- Middleware для auth, rate limit, idempotency, request ID.
- `ThrottleRequests`/rate limiter для лимитов.
- Queues для async processing и webhook delivery.
- Events/listeners для интеграционных событий, но не путать с доменными инвариантами.

Практические замечания:

- Не отдавать Eloquent models напрямую наружу; использовать Resource/DTO.
- Не держать бизнес-логику в Controller.
- Валидацию request отделять от доменной проверки.
- Для idempotency использовать отдельную таблицу с unique index и транзакциями.
- Для optimistic locking добавить `version` column или ETag на основе version.
- Исключения маппить в единый error format через exception handler.

## Symfony API notes

Полезные элементы Symfony:

- Routing attributes/YAML/XML.
- Serializer + DTO/View models.
- Validator component.
- Security voters/access control.
- Event subscribers/listeners для cross-cutting concerns.
- Messenger для async commands/events.
- RateLimiter component.
- API Platform, если подходит resource-driven подход.

Практические замечания:

- Не отдавать Doctrine entities как внешний контракт без слоя нормализации.
- Разделять Input DTO, Command и Output DTO.
- Constraint validation не заменяет доменные инварианты.
- В API Platform внимательно контролировать exposed operations, serialization groups и security expressions.
- Problem Details можно централизовать через exception listener/subscriber.

## Common pitfalls и tradeoffs

- Чрезмерная REST-чистота против бизнес-практичности: иногда command resource проще и понятнее.
- Generic endpoints вроде `/search` могут быть нормальны, если поиск - самостоятельный capability.
- GraphQL лучше для гибкого чтения сложных графов, но не отменяет идемпотентность команд и auth complexity.
- gRPC удобен для internal service-to-service, но хуже для публичных browser/client integrations.
- Async `202 Accepted` лучше, чем держать HTTP-запрос 60 секунд.
- Strong consistency удобна, но иногда expensive; eventual consistency нужно явно отражать в API.
- Public API требует более строгой политики совместимости, чем internal API.

## Senior-level short answers

### Что такое хороший API?

Хороший API стабилен, предсказуем, документирован, безопасен, наблюдаем и отражает сценарии клиента, а не структуру БД.

### REST vs RPC?

REST хорош для ресурсной модели и стандартной HTTP-семантики. RPC проще для команд и сложных операций. Я выбираю стиль по домену и клиентам, но всегда фиксирую контракт, ошибки, auth, retries и compatibility.

### Как проектировать создание платежа?

`POST /payments` с обязательным `Idempotency-Key`, валидацией amount/currency/order, транзакционной записью операции, уникальными constraints, асинхронной обработкой при необходимости, `201` или `202`, стабильным error format и безопасными retries.

### Как избежать lost updates?

Использовать optimistic locking: ETag/`If-Match` или `version` field. При несовпадении версии вернуть `412 Precondition Failed` или `409 Conflict` в зависимости от выбранного контракта.

### Когда нужна новая версия API?

Когда изменение ломает существующих клиентов: удаление/переименование поля, изменение типа/семантики, новые required поля, изменение auth, error codes или pagination contract.

### Что должно быть в OpenAPI?

Не только happy path. Нужны схемы request/response, все значимые status codes, errors, auth, headers, examples, pagination, rate limits, idempotency и deprecation.

## Мини-практика

### Спроектировать endpoint создания заказа

Проверь себя:

- Какой метод и URL?
- Нужен ли `Idempotency-Key`?
- Какой status code при синхронном создании?
- Что вернуть при duplicate request?
- Какие ошибки валидации возможны?
- Какие business conflicts возможны?
- Что будет при timeout клиента?
- Нужно ли событие/webhook?

Ожидаемый скелет:

```http
POST /orders
Idempotency-Key: <uuid-or-ulid>
Content-Type: application/json
```

Ответы:

- `201 Created` + `Location` при создании.
- Повтор с тем же key и payload - тот же response.
- Повтор с тем же key и другим payload - `409 Conflict`.
- Ошибки полей - `422`.
- Конфликт состояния, например товар недоступен - `409`.

### Спроектировать PATCH профиля

Проверь себя:

- Чем `PATCH` лучше `PUT`?
- Что делать с `null`?
- Как защититься от lost update?
- Что вернуть, если email уже занят?
- Какие поля нельзя менять клиенту?

Ожидаемый ответ:

- `PATCH /profile` или `PATCH /users/{id}`.
- Явно документировать semantics отсутствующего поля и `null`.
- Использовать ETag/`If-Match` для конкурентных изменений.
- Email conflict вернуть `409` или `422` по принятому контракту.
- Internal/system fields не принимать в request.

## Чеклисты

### API endpoint checklist

- Ресурс и use case понятны клиенту.
- Метод соответствует HTTP-семантике.
- Status codes описаны для success и errors.
- Request/response schemas стабильны.
- Error format единый.
- Валидация и business conflicts разделены.
- Идемпотентность продумана.
- Retries/timeouts описаны.
- Auth и permissions определены.
- Pagination/filtering/sorting ограничены и индексируемы.
- Rate limits и quotas понятны.
- OpenAPI обновлен.
- Логи, request ID, metrics, tracing есть.

### Compatibility checklist

- Изменение additive?
- Нет удаления или переименования полей?
- Типы и enum values не ломают клиентов?
- Error codes не изменили смысл?
- Есть deprecation policy для старого контракта?
- Есть метрики использования старой версии?
- Есть contract tests?

### Idempotency checklist

- Для опасного `POST` есть `Idempotency-Key`.
- Key уникален на scope клиента/tenant.
- Payload hash сохраняется.
- Запись защищена unique index.
- Параллельные повторы корректны.
- Response кешируется или восстанавливается.
- TTL определен.
- Поведение для `processing`, success, failure документировано.

## Вопросы для самопроверки

- Чем safe метод отличается от idempotent метода?
- Почему `DELETE` считается идемпотентным?
- Когда `POST` можно безопасно ретраить?
- Чем `409` отличается от `422`?
- Какой формат ошибки удобен для публичного API?
- Что такое ETag и как он помогает при concurrent update?
- Какие изменения API являются breaking?
- Почему добавление enum value может сломать клиента?
- Какие плюсы и минусы у cursor pagination?
- Как проектировать webhook delivery при at-least-once semantics?
- Почему нельзя отдавать ORM entity напрямую?
- Что обязательно описать в OpenAPI, кроме happy path?

## Ссылки

- RFC 9110: HTTP Semantics - https://www.rfc-editor.org/rfc/rfc9110
- RFC 9111: HTTP Caching - https://www.rfc-editor.org/rfc/rfc9111
- RFC 9457: Problem Details for HTTP APIs - https://www.rfc-editor.org/rfc/rfc9457
- RFC 5789: PATCH Method for HTTP - https://www.rfc-editor.org/rfc/rfc5789
- RFC 7396: JSON Merge Patch - https://www.rfc-editor.org/rfc/rfc7396
- RFC 6902: JSON Patch - https://www.rfc-editor.org/rfc/rfc6902
- MDN HTTP response status codes - https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status
- MDN HTTP request methods - https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Methods
- OpenAPI Specification - https://spec.openapis.org/oas/latest.html
- Swagger documentation - https://swagger.io/docs/specification/
- Microsoft REST API Guidelines - https://github.com/microsoft/api-guidelines
- Google API Improvement Proposals - https://google.aip.dev/
- Zalando RESTful API Guidelines - https://opensource.zalando.com/restful-api-guidelines/
- Stripe Idempotent requests - https://docs.stripe.com/api/idempotent_requests
- Stripe API versioning - https://docs.stripe.com/api/versioning
- IETF RateLimit headers RFC 9333 - https://www.rfc-editor.org/rfc/rfc9333
