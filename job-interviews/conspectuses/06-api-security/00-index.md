# API и Security

Раздел объединяет конспекты по API design, REST/OpenAPI, идемпотентности, версионированию и backend security. Материал перегруппирован из двух старых файлов так, чтобы пересечения не дублировались дословно: API-файлы отвечают за контракт и HTTP-семантику, security-файлы - за угрозы, auth, токены, секреты и операционные практики.

## Как читать

1. Начать с API design: ресурсы, HTTP-методы, статусы, ошибки, pagination и OpenAPI.
2. Затем пройти надежность API: идемпотентность, retries, optimistic locking, versioning, deprecation, webhooks и rate limits.
3. После этого читать backend security: OWASP Top 10, SQLi, XSS, CSRF, SSRF, CORS, uploads, TLS и secure headers.
4. В конце пройти auth и secrets: sessions, cookies, OAuth2/OIDC, JWT, RBAC/ABAC, secrets, sensitive logging и supply chain.
5. Использовать coverage map для проверки, что темы из старых конспектов не потеряны.

## Файлы

- [01 API Design REST OpenAPI](01-api-design-rest-openapi.md)
- [02 API Reliability Versioning Webhooks](02-api-reliability-versioning-webhooks.md)
- [03 Backend Security OWASP](03-backend-security-owasp.md)
- [04 Auth OAuth JWT Secrets](04-auth-oauth-jwt-secrets.md)
- [99 Coverage map](99-coverage-map.md)

## Сквозные связи

- `401`/`403`, auth schemes и scopes: кратко в [01](01-api-design-rest-openapi.md), глубоко в [04](04-auth-oauth-jwt-secrets.md).
- Idempotency, retries и duplicate prevention: API-контракт в [02](02-api-reliability-versioning-webhooks.md), abuse/security сценарии в [03](03-backend-security-owasp.md) и [04](04-auth-oauth-jwt-secrets.md).
- Webhooks: delivery contract в [02](02-api-reliability-versioning-webhooks.md), прием и проверка подписи в [03](03-backend-security-owasp.md).
- Rate limiting: HTTP-контракт и headers в [02](02-api-reliability-versioning-webhooks.md), abuse protection в [03](03-backend-security-owasp.md).
- Sensitive data: не раскрывать внутренности API в [01](01-api-design-rest-openapi.md), не логировать secrets/PII в [04](04-auth-oauth-jwt-secrets.md).
