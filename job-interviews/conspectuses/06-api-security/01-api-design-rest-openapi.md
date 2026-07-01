# API Design, REST и OpenAPI

<!-- markdownlint-disable MD013 -->

Гайд для подготовки Lead/Senior PHP backend developer к интервью. Фокус: как проектировать стабильный HTTP API-контракт, объяснять REST-компромиссы, выбирать статусы, описывать ошибки и фиксировать контракт в OpenAPI.

## Быстрая карта темы

- API design - это контракт между клиентом и сервером, а не набор роутов поверх таблиц.
- REST - архитектурный стиль с ресурсами, стандартной HTTP-семантикой и stateless-взаимодействием.
- OpenAPI - машинно-читаемый контракт для документации, генерации клиентов, моков, contract testing и review изменений.
- Хороший API стабилен, предсказуем, документирован, безопасен, наблюдаем и отражает сценарии клиента.
- Security-схемы, rate limits и idempotency headers должны быть частью публичного контракта, а не устной договоренностью.

Senior-level ответ:

> Хороший API - это стабильный контракт вокруг клиентских сценариев: понятные ресурсы, корректная HTTP-семантика, единый формат ошибок, документированные ограничения, auth, retries, observability и политика совместимости.

## REST: ограничения и смысл

REST задает не стиль URL, а набор architectural constraints.

| Constraint | Смысл для backend API |
|---|---|
| Client-server | UI и сервер разделены, клиент не зависит от внутренней модели БД. |
| Stateless | Каждый запрос содержит все данные для обработки, сервер не хранит скрытое состояние запроса. |
| Cacheable | Ответы явно говорят, можно ли их кешировать. |
| Uniform interface | Ресурсы, HTTP-методы, media types, links и status codes используются единообразно. |
| Layered system | Клиенту не важно, отвечает origin server, gateway, proxy или CDN. |
| Code on demand | Необязательное ограничение, для backend API обычно вторично. |

Частые ошибки:

- Называть REST любой JSON-over-HTTP API.
- Делать `POST /getUser` вместо `GET /users/{id}`.
- Всегда возвращать `200 OK`, даже при ошибках.
- Хранить серверное состояние между запросами без явной модели.
- Проектировать внешний контракт из структуры таблиц, а не из домена и сценариев клиента.

## Ресурсы и URL

Ресурс - доменная сущность или коллекция, доступная по стабильному идентификатору.

```http
GET /users
GET /users/{userId}
GET /users/{userId}/orders
POST /orders
PATCH /orders/{orderId}
```

Рекомендации:

- Использовать существительные во множественном числе: `/users`, `/orders`.
- Не раскрывать внутренние ID, если это риск; можно использовать UUID, ULID или public ID.
- Не делать глубокую вложенность без необходимости: `/users/{id}/orders/{orderId}/items/{itemId}` быстро становится хрупким.
- Подресурс оправдан, если он существует только в контексте родителя или часто выбирается в этом контексте.
- Действия моделировать как ресурсы, если они имеют состояние: `POST /payments/{id}/captures`, `POST /orders/{id}/cancellations`.

Компромисс:

- Чистый REST удобен для CRUD и доменных ресурсов.
- Для сложных команд иногда практичнее command endpoint: `POST /orders/{id}:cancel` или `POST /order-cancellations`.
- Главное - явно описать контракт, идемпотентность, ошибки, auth и retries.

## HTTP-методы

| Метод | Назначение | Safe | Idempotent | Типичный body |
|---|---|---|---|---|
| `GET` | Получить ресурс | Да | Да | Нет |
| `HEAD` | Метаданные без body | Да | Да | Нет |
| `OPTIONS` | Доступные методы/CORS | Да | Да | Нет |
| `POST` | Создать ресурс или выполнить команду | Нет | Нет по умолчанию | Да |
| `PUT` | Полная замена ресурса по известному URI | Нет | Да | Да |
| `PATCH` | Частичное изменение ресурса | Нет | Не всегда | Да |
| `DELETE` | Удалить ресурс | Нет | Да по семантике | Обычно нет |

Важно:

- Idempotent не значит "без побочных эффектов". `DELETE /users/1` может удалить пользователя, но повторный `DELETE` не должен удалить что-то еще.
- `POST` можно сделать идемпотентным через `Idempotency-Key`.
- `PATCH` может быть идемпотентным, если формат операции идемпотентен, например `{"status":"cancelled"}`.
- `PATCH` может быть неидемпотентным, если операция накопительная, например `{"increment":1}`.

## PUT vs PATCH

`PUT`:

- Клиент отправляет полное представление ресурса.
- Повтор запроса приводит к тому же состоянию.
- Риск: случайно затереть поля, неизвестные клиенту.

`PATCH`:

- Клиент отправляет изменения.
- Удобен для частичного обновления.
- Нужно выбрать формат: JSON Merge Patch или JSON Patch.
- Нужно явно описать семантику absent fields и `null`.

Senior-level ответ:

> Я использую `PUT`, когда клиент владеет полным представлением ресурса, и `PATCH`, когда обновляет ограниченный набор полей. Для конкурентных обновлений добавляю `If-Match` с ETag или version field.

## HTTP-статусы

Успешные ответы:

| Код | Когда использовать |
|---|---|
| `200 OK` | Успешный запрос с body. |
| `201 Created` | Ресурс создан, желательно вернуть `Location`. |
| `202 Accepted` | Запрос принят на асинхронную обработку. |
| `204 No Content` | Успех без body, часто `DELETE` или `PATCH`. |

Ошибки клиента:

| Код | Когда использовать |
|---|---|
| `400 Bad Request` | Некорректный синтаксис или общий плохой запрос. |
| `401 Unauthorized` | Нет или неверная аутентификация. |
| `403 Forbidden` | Аутентифицирован, но нет прав. |
| `404 Not Found` | Ресурс не найден или скрыт политикой доступа. |
| `405 Method Not Allowed` | Метод не поддержан для ресурса. |
| `409 Conflict` | Конфликт состояния, duplicate или business conflict. |
| `412 Precondition Failed` | Не прошел `If-Match`/`If-Unmodified-Since`. |
| `415 Unsupported Media Type` | Неподдерживаемый `Content-Type`. |
| `422 Unprocessable Content` | Валидный JSON, но ошибки доменной/полевой валидации. |
| `429 Too Many Requests` | Rate limit превышен. |

Ошибки сервера:

| Код | Когда использовать |
|---|---|
| `500 Internal Server Error` | Неожиданная ошибка. |
| `502 Bad Gateway` | Ошибка upstream-сервиса через gateway. |
| `503 Service Unavailable` | Сервис временно недоступен. |
| `504 Gateway Timeout` | Timeout upstream-сервиса. |

Pitfalls:

- `401` vs `403`: `401` про аутентификацию, `403` про авторизацию.
- `400` vs `422`: `400` для плохого формата, `422` для семантической валидации.
- `409` vs `422`: `409` для конфликта текущего состояния системы, `422` для неправильных входных данных.
- Не возвращать stack trace, SQL errors, class names и внутренние детали во внешнее API.

## Request/response design

Общие принципы:

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

## Envelope или bare resource

Bare resource проще и ближе к HTTP:

```json
{
  "id": "usr_123",
  "email": "user@example.com"
}
```

Envelope удобен для `meta`, pagination, tracing и uniform error handling:

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

Главное - единообразие внутри API.

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

Offset pagination:

```http
GET /orders?limit=50&offset=100
```

Плюсы: просто реализовать, удобно для админок и страниц.

Минусы: медленно на больших offset, нестабильно при вставках/удалениях между запросами.

Cursor pagination:

```http
GET /orders?limit=50&cursor=eyJjcmVhdGVkX2F0Ijoi..."
```

Плюсы: лучше для больших списков, стабильнее при изменениях данных.

Минусы: сложнее UX для произвольного перехода на страницу, нужна стабильная сортировка вроде `created_at,id`.

Filtering/sorting:

```http
GET /orders?status=paid&created_from=2026-01-01&sort=-created_at,id
```

Рекомендации:

- Белый список полей фильтрации и сортировки.
- Ограничивать `limit` максимальным значением.
- Индексировать популярные фильтры.
- Документировать порядок сортировки и tie-breaker.
- Не давать клиенту произвольный SQL-like query без sandbox и лимитов.

## Authentication и authorization в API-контракте

Authentication отвечает на вопрос "кто ты?", authorization - "что тебе можно?".

Типовые схемы:

- Session cookie: удобно для browser apps, нужен CSRF protection.
- Bearer token/JWT: удобно для API, важно ограничивать TTL и scopes.
- OAuth 2.0/OIDC: стандарт для delegated access и SSO.
- API keys: просты для server-to-server, требуют rotation, scopes, audit.
- mTLS: сильная machine-to-machine аутентификация.

Рекомендации для API design:

- Всегда HTTPS.
- Не передавать токены в query string.
- Использовать scopes/permissions и документировать их в OpenAPI.
- Логировать `subject`, `client_id`, `scope`, но не секреты.
- Делать key rotation и revocation.
- Для multi-tenant API проверять tenant boundary на каждом запросе.

Pitfalls:

- Проверять только наличие JWT, но не `aud`, `iss`, `exp`, signature.
- Полагаться на client-side authorization.
- Смешивать user permissions и service permissions.
- Возвращать `403` с подробностями, раскрывающими существование чужого ресурса.

Подробности OAuth2/OIDC/JWT, sessions, cookies и secrets находятся в [04 Auth OAuth JWT Secrets](04-auth-oauth-jwt-secrets.md).

## OpenAPI / Swagger

OpenAPI - спецификация HTTP API: endpoints, schemas, auth, параметры, ответы, ошибки и examples.

Что документировать:

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
- Code-first: генерировать OpenAPI из атрибутов/аннотаций. Быстрее, но легко получить документацию реализации, а не контракта.
- Проверять OpenAPI в CI: lint, breaking changes, schema validation.
- Использовать examples как часть документации и тестов.

Pitfalls:

- Документация не соответствует реальности.
- Описан только `200`, нет ошибок.
- Нет примеров.
- Все поля `string`, constraints не заданы.
- OpenAPI генерируется, но не проходит review как контракт.

Senior-level ответ:

> В OpenAPI должны быть не только happy path: нужны схемы request/response, все значимые status codes, errors, auth, headers, examples, pagination, rate limits, idempotency и deprecation.

## SOAP и legacy integrations

SOAP все еще встречается в enterprise, банках и госинтеграциях.

Что важно знать:

- Контракт описывается WSDL.
- Сообщения XML, часто строгие XSD schemas.
- Ошибки через SOAP Fault.
- WS-Security может включать подписи, encryption и timestamps.
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

## Чеклист API endpoint

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

## Вопросы для самопроверки

- Чем safe метод отличается от idempotent метода?
- Почему `DELETE` считается идемпотентным?
- Чем `409` отличается от `422`?
- Какой формат ошибки удобен для публичного API?
- Какие плюсы и минусы у cursor pagination?
- Почему нельзя отдавать ORM entity напрямую?
- Что обязательно описать в OpenAPI, кроме happy path?

## Дополнительное чтение

- RFC 9110: HTTP Semantics - https://www.rfc-editor.org/rfc/rfc9110
- RFC 9111: HTTP Caching - https://www.rfc-editor.org/rfc/rfc9111
- RFC 9457: Problem Details for HTTP APIs - https://www.rfc-editor.org/rfc/rfc9457
- RFC 5789: PATCH Method for HTTP - https://www.rfc-editor.org/rfc/rfc5789
- RFC 7396: JSON Merge Patch - https://www.rfc-editor.org/rfc/rfc7396
- RFC 6902: JSON Patch - https://www.rfc-editor.org/rfc/rfc6902
- MDN HTTP response status codes - https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status
- MDN HTTP request methods - https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Methods
- MDN HTTP authentication - https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Authentication
- OpenAPI Specification - https://spec.openapis.org/oas/latest.html
- Swagger documentation - https://swagger.io/docs/specification/
- Microsoft REST API Guidelines - https://github.com/microsoft/api-guidelines
- Google API Improvement Proposals - https://google.aip.dev/
- Zalando RESTful API Guidelines - https://opensource.zalando.com/restful-api-guidelines/
- API Platform documentation - https://api-platform.com/docs/
- JSON:API specification - https://jsonapi.org/format/
