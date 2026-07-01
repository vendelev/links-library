# API Reliability, Versioning и Webhooks

<!-- markdownlint-disable MD013 -->

Конспект про надежность API-контракта: идемпотентность, retries, timeouts, optimistic locking, compatibility, deprecation, webhooks и rate limiting.

## Идемпотентность

Идемпотентность означает, что повтор одного и того же запроса дает тот же итоговый эффект. Это критично для платежей, создания заказов, внешних интеграций и retries.

`POST` по умолчанию не идемпотентен, но его можно сделать идемпотентным через `Idempotency-Key`.

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
- `status`: `processing`, `succeeded`, `failed`.
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

- Retry безопасен для `GET`, `HEAD`, `PUT`, `DELETE` по HTTP-семантике, но backend side effects все равно нужно учитывать.
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

Senior-level ответ:

> Чтобы избежать lost update, использую optimistic locking: ETag/`If-Match` или `version` field. При несовпадении версии возвращаю `412 Precondition Failed` или `409 Conflict` в зависимости от контракта.

## API versioning

Версионирование нужно для breaking changes. Не каждое изменение требует новой версии.

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

## Breaking vs non-breaking changes

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

- Добавление enum value может быть breaking для клиентов с exhaustive switch.
- В публичном API надо документировать, что клиенты должны быть tolerant к unknown values.

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

## Webhooks

Webhook - обратный вызов от сервера к клиенту при событии.

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

Правила отправителя:

- Подписывать payload HMAC или asymmetric signature.
- Передавать timestamp и защищаться от replay attacks.
- Делать delivery at-least-once, поэтому получатель должен быть идемпотентным.
- Ретраить с backoff.
- Давать dashboard/log delivery attempts.
- Не требовать мгновенного ответа; клиент должен быстро вернуть `2xx` и обработать async.
- Версионировать event schema.

Правила получателя:

- Проверять подпись webhook body shared secret'ом или asymmetric signature.
- Проверять timestamp и replay window.
- Использовать idempotency key/event ID.
- Не доверять IP allowlist как единственной защите.
- Логировать verification failures без тела с секретными данными.

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

Практика API-контракта:

- Лимитировать по API key/user/IP/tenant в зависимости от продукта.
- Разделять read/write limits.
- Для дорогих endpoints вводить отдельные quotas.
- Возвращать `429 Too Many Requests`.
- Документировать retry behavior.

Abuse protection детали находятся в [03 Backend Security OWASP](03-backend-security-owasp.md).

## Чеклист compatibility

- Изменение additive?
- Нет удаления или переименования полей?
- Типы и enum values не ломают клиентов?
- Error codes не изменили смысл?
- Есть deprecation policy для старого контракта?
- Есть метрики использования старой версии?
- Есть contract tests?

## Чеклист idempotency

- Для опасного `POST` есть `Idempotency-Key`.
- Key уникален на scope клиента/tenant.
- Payload hash сохраняется.
- Запись защищена unique index.
- Параллельные повторы корректны.
- Response кешируется или восстанавливается.
- TTL определен.
- Поведение для `processing`, success, failure документировано.

## Вопросы для самопроверки

- Когда `POST` можно безопасно ретраить?
- Что такое ETag и как он помогает при concurrent update?
- Какие изменения API являются breaking?
- Почему добавление enum value может сломать клиента?
- Как проектировать webhook delivery при at-least-once semantics?
- Что вернуть при повторе `Idempotency-Key` с другим payload?
- Как выбрать TTL для idempotency keys?

## Дополнительное чтение

- Stripe Idempotent requests - https://docs.stripe.com/api/idempotent_requests
- Stripe API versioning - https://docs.stripe.com/api/versioning
- Brandur Leach: Designing robust and predictable APIs with idempotency - https://stripe.com/blog/idempotency
- RFC 9110: Conditional requests and HTTP semantics - https://www.rfc-editor.org/rfc/rfc9110
- RFC 9333: RateLimit headers - https://www.rfc-editor.org/rfc/rfc9333
- IETF draft: The Deprecation HTTP Header Field - https://datatracker.ietf.org/doc/html/draft-ietf-httpapi-deprecation-header
- RFC 8594: The Sunset HTTP Header Field - https://datatracker.ietf.org/doc/html/rfc8594
- Pact consumer-driven contract testing - https://docs.pact.io/
- Martin Fowler: Consumer-Driven Contracts - https://martinfowler.com/articles/consumerDrivenContracts.html
- GitHub Webhooks docs - https://docs.github.com/en/webhooks
- Stripe Webhooks docs - https://docs.stripe.com/webhooks
- Slack: Verifying requests from Slack - https://api.slack.com/authentication/verifying-requests-from-slack
