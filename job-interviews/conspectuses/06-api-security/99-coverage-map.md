# Coverage Map

<!-- markdownlint-disable MD013 -->

Карта проверки переноса информации из исходных конспектов в новые файлы `job-interviews/conspectuses/06-api-security`.

## Исходные файлы

1. `job-interviews/API Design REST OpenAPI идемпотентность версионирование.md`
2. `job-interviews/Backend Security OWASP OAuth JWT Secrets.md`

## Новые файлы

1. `00-index.md`
2. `01-api-design-rest-openapi.md`
3. `02-api-reliability-versioning-webhooks.md`
4. `03-backend-security-owasp.md`
5. `04-auth-oauth-jwt-secrets.md`

## Что куда перенесено

| Тема | Старые файлы | Новый основной файл | Статус |
|---|---|---|---|
| Быстрая карта API design, REST, OpenAPI, идемпотентность, версионирование | API Design | `00-index.md`, `01-api-design-rest-openapi.md`, `02-api-reliability-versioning-webhooks.md` | перенесено |
| REST constraints: client-server, stateless, cacheable, uniform interface, layered system, code on demand | API Design | `01-api-design-rest-openapi.md` | перенесено |
| REST pitfalls: JSON-over-HTTP, `POST /getUser`, always `200`, hidden server state, DB-first API | API Design | `01-api-design-rest-openapi.md` | перенесено |
| Ресурсы и URL, plural nouns, public IDs, nesting, subresources, command resources | API Design | `01-api-design-rest-openapi.md` | перенесено |
| HTTP methods, safe/idempotent, POST idempotency, PATCH idempotency | API Design | `01-api-design-rest-openapi.md` | перенесено |
| PUT vs PATCH, JSON Merge Patch, JSON Patch, ETag/If-Match | API Design | `01-api-design-rest-openapi.md`, `02-api-reliability-versioning-webhooks.md` | перенесено |
| HTTP success/client/server status codes | API Design | `01-api-design-rest-openapi.md` | перенесено |
| `401` vs `403`, `400` vs `422`, `409` vs `422`, no stack traces | API Design, Backend Security | `01-api-design-rest-openapi.md`, `03-backend-security-owasp.md` | перенесено |
| Request/response design: naming, dates, money, enums, nullable/absent, no internal fields | API Design | `01-api-design-rest-openapi.md` | перенесено |
| Envelope vs bare resource | API Design | `01-api-design-rest-openapi.md` | перенесено |
| Error format: RFC 9457 Problem Details, stable codes, request_id/trace_id | API Design | `01-api-design-rest-openapi.md` | перенесено |
| Validation levels: transport/schema, application, domain, persistence constraints | API Design | `01-api-design-rest-openapi.md` | перенесено |
| Pagination: offset, cursor, filtering, sorting, allowlist, limits, indexes | API Design | `01-api-design-rest-openapi.md` | перенесено |
| Idempotency-Key: request hash, status, response cache, TTL, unique constraint, processing state | API Design | `02-api-reliability-versioning-webhooks.md` | перенесено |
| Retries/timeouts: safe methods, POST with Idempotency-Key, backoff+jitter, Retry-After, timeout types | API Design | `02-api-reliability-versioning-webhooks.md` | перенесено |
| Optimistic locking, ETag, conditional requests, lost update, `412 Precondition Failed` | API Design | `02-api-reliability-versioning-webhooks.md` | перенесено |
| API versioning: URI, header, media type, date-based | API Design | `02-api-reliability-versioning-webhooks.md` | перенесено |
| Breaking vs non-breaking changes, enum expansion pitfall | API Design | `02-api-reliability-versioning-webhooks.md` | перенесено |
| Backward compatibility rules, additive changes, tolerant clients, contract tests | API Design | `02-api-reliability-versioning-webhooks.md` | перенесено |
| Deprecation policy, Sunset/Deprecation/Link headers, metrics and communication | API Design | `02-api-reliability-versioning-webhooks.md` | перенесено |
| OpenAPI/Swagger: paths, schemas, errors, auth, examples, lint, breaking changes | API Design | `01-api-design-rest-openapi.md` | перенесено |
| Webhooks: signature, timestamp, replay, at-least-once, retries, dashboard, versioned events | API Design, Backend Security | `02-api-reliability-versioning-webhooks.md`, `03-backend-security-owasp.md` | перенесено |
| Rate limiting: goals, algorithms, headers, scopes, quotas, `429` | API Design, Backend Security | `02-api-reliability-versioning-webhooks.md`, `03-backend-security-owasp.md` | перенесено |
| API authentication schemes: session cookie, Bearer/JWT, OAuth2/OIDC, API keys, mTLS | API Design, Backend Security | `01-api-design-rest-openapi.md`, `04-auth-oauth-jwt-secrets.md` | перенесено |
| API auth pitfalls: token in query, JWT claims, client-side authz, tenant boundary, `403` disclosure | API Design, Backend Security | `01-api-design-rest-openapi.md`, `04-auth-oauth-jwt-secrets.md` | перенесено |
| SOAP and legacy integrations: WSDL, XSD, SOAP Fault, WS-Security, MTOM, adapter layer | API Design | `01-api-design-rest-openapi.md` | перенесено |
| Laravel API notes: routes, Form Request, API Resources, Policies, middleware, rate limiter, queues | API Design, Backend Security | `01-api-design-rest-openapi.md`, `03-backend-security-owasp.md` | перенесено |
| Symfony API notes: routing, serializer, validator, voters, Messenger, RateLimiter, API Platform | API Design, Backend Security | `01-api-design-rest-openapi.md`, `03-backend-security-owasp.md` | перенесено |
| API tradeoffs: REST vs RPC, GraphQL, gRPC, async `202`, consistency, public vs internal API | API Design | `01-api-design-rest-openapi.md` | перенесено |
| API senior answers, endpoint creation practice, PATCH profile practice, API endpoint checklist | API Design | `01-api-design-rest-openapi.md`, `02-api-reliability-versioning-webhooks.md` | перенесено |
| API self-check questions | API Design | `01-api-design-rest-openapi.md`, `02-api-reliability-versioning-webhooks.md` | перенесено |
| Как отвечать на security interview: threat model, prevention/detection/response, protocols, tradeoffs | Backend Security | `03-backend-security-owasp.md` | перенесено |
| OWASP Top 10 A01-A10 | Backend Security | `03-backend-security-owasp.md` | перенесено |
| Broken Access Control, IDOR, object-level authorization, mass assignment, Laravel/Symfony authorization | Backend Security | `03-backend-security-owasp.md`, `04-auth-oauth-jwt-secrets.md` | перенесено |
| Cryptographic Failures, password hashing, TLS, envelope encryption/KMS, no custom crypto | Backend Security | `03-backend-security-owasp.md`, `04-auth-oauth-jwt-secrets.md` | перенесено |
| Injection, SQL/NoSQL/OS/LDAP/template, prepared statements, allowlist, shell isolation | Backend Security | `03-backend-security-owasp.md` | перенесено |
| Security Misconfiguration: debug, headers, CORS, buckets, default passwords, `.env` | Backend Security | `03-backend-security-owasp.md`, `04-auth-oauth-jwt-secrets.md` | перенесено |
| Vulnerable/outdated components and software/data integrity failures | Backend Security | `04-auth-oauth-jwt-secrets.md` | перенесено |
| Identification/authentication failures: weak passwords, credential stuffing, session fixation, reset password, MFA | Backend Security | `03-backend-security-owasp.md`, `04-auth-oauth-jwt-secrets.md` | перенесено |
| Security logging and monitoring failures, audit trail, auth events, permission denials | Backend Security | `03-backend-security-owasp.md`, `04-auth-oauth-jwt-secrets.md` | перенесено |
| SQL Injection details and PHP example | Backend Security | `03-backend-security-owasp.md` | перенесено |
| XSS: stored/reflected/DOM, escaping, Blade/Twig, sanitizer, CSP, JSON in HTML | Backend Security | `03-backend-security-owasp.md` | перенесено |
| CSRF: tokens, SameSite, Origin/Referer, no state change via GET, Bearer-token model | Backend Security | `03-backend-security-owasp.md` | перенесено |
| SSRF: webhooks/preview/import/PDF/integrations, DNS/private ranges/redirects/egress firewall | Backend Security | `03-backend-security-owasp.md` | перенесено |
| Authentication vs Authorization | Backend Security, API Design | `01-api-design-rest-openapi.md`, `04-auth-oauth-jwt-secrets.md` | перенесено |
| Sessions and Cookies: flags, rotation, invalidation, timeouts, server-side storage, Laravel/Symfony config | Backend Security | `04-auth-oauth-jwt-secrets.md` | перенесено |
| Password Hashing: Argon2id, bcrypt, salt, pepper, rehash, PHP/Laravel/Symfony examples | Backend Security | `04-auth-oauth-jwt-secrets.md` | перенесено |
| OAuth2/OIDC basics, roles, flows, tokens, PKCE, implicit/password legacy | Backend Security | `04-auth-oauth-jwt-secrets.md` | перенесено |
| JWT validation, `alg=none`, `kid`, TTL, revocation, key rotation, payload minimization | Backend Security | `04-auth-oauth-jwt-secrets.md` | перенесено |
| Access/refresh tokens, refresh token rotation, browser storage tradeoffs | Backend Security | `04-auth-oauth-jwt-secrets.md` | перенесено |
| RBAC/ABAC and hybrid object-level policies | Backend Security | `04-auth-oauth-jwt-secrets.md` | перенесено |
| CORS: concrete origin, credentials, preflight, not authz, pitfalls | Backend Security | `03-backend-security-owasp.md` | перенесено |
| File upload security: RCE, SVG/HTML XSS, malware, zip bomb, traversal, metadata, checks | Backend Security | `03-backend-security-owasp.md` | перенесено |
| Secrets management: DB passwords, API keys, JWT keys, OAuth secrets, webhook secrets, rotation | Backend Security | `04-auth-oauth-jwt-secrets.md` | перенесено |
| Composer/npm supply-chain security, CI/CD permissions, Dependabot/Renovate | Backend Security | `04-auth-oauth-jwt-secrets.md` | перенесено |
| Logging sensitive data: no passwords/tokens/headers/PII, redaction, retention, tamper resistance | Backend Security | `04-auth-oauth-jwt-secrets.md` | перенесено |
| Secure headers and TLS basics | Backend Security | `03-backend-security-owasp.md` | перенесено |
| PHP/Laravel/Symfony security mini-checklist | Backend Security | `03-backend-security-owasp.md` | перенесено |
| Practical scenarios: protect `GET /invoices/{id}`, JWT vs session, refresh token rotation, webhook receiving | Backend Security | `03-backend-security-owasp.md`, `04-auth-oauth-jwt-secrets.md` | перенесено |
| Частые ошибки кандидатов | Backend Security | `03-backend-security-owasp.md`, `04-auth-oauth-jwt-secrets.md` | перенесено |
| Security review practice, JWT validation checklist, secrets incident checklist | Backend Security | `03-backend-security-owasp.md`, `04-auth-oauth-jwt-secrets.md` | перенесено |
| Security self-check questions | Backend Security | `03-backend-security-owasp.md`, `04-auth-oauth-jwt-secrets.md` | перенесено |

## Что было намеренно не продублировано дословно

Новые файлы не являются побайтовым копированием исходников. Повторы были объединены, а темы разнесены по основному месту ответственности.

Сокращены без потери смысла:

- повторные объяснения authentication/authorization: краткий API-контракт оставлен в `01-api-design-rest-openapi.md`, подробности перенесены в `04-auth-oauth-jwt-secrets.md`;
- повторные объяснения rate limiting: HTTP headers и контракт в `02-api-reliability-versioning-webhooks.md`, abuse scenarios в `03-backend-security-owasp.md`;
- повторные объяснения webhook signatures/idempotency: delivery contract в `02-api-reliability-versioning-webhooks.md`, прием webhook в `03-backend-security-owasp.md`;
- повторные предупреждения про secrets/PII/logging: короткие упоминания сохранены в API/security контекстах, полный материал в `04-auth-oauth-jwt-secrets.md`;
- ссылки из исходных файлов сохранены и расширены в разделах `Дополнительное чтение`.

## Проверка ссылок на источники

Ссылки из исходников перенесены по темам:

- HTTP, caching, Problem Details, PATCH, JSON Merge Patch, JSON Patch, MDN, OpenAPI, Swagger, Microsoft/Google/Zalando API guidelines - `01-api-design-rest-openapi.md`.
- Stripe idempotency/versioning, RateLimit headers, Deprecation/Sunset headers, Pact, webhooks - `02-api-reliability-versioning-webhooks.md`.
- OWASP Top 10, Cheat Sheet Series, SQLi, XSS, CSRF, SSRF, file uploads, CSP, secure headers, CORS, Laravel/Symfony security - `03-backend-security-owasp.md`.
- OAuth2, OAuth security BCP, OIDC, JWT, password hashing, authentication/authorization/secrets/logging cheat sheets, Laravel/Symfony auth, Composer/npm audit - `04-auth-oauth-jwt-secrets.md`.

## Итог проверки покрытия

Информация из двух исходных файлов перенесена в новую структуру по смыслу. Явных потерянных тем не осталось: все крупные разделы, примеры, pitfalls, senior answers, checklists, practice tasks, self-check questions и ссылки распределены по новым файлам.
