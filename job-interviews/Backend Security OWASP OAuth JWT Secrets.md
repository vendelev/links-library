# Backend Security: OWASP, OAuth, JWT, Secrets

Гайд для Lead/Senior PHP backend developer перед интервью. Фокус: практическое объяснение рисков, типовые ошибки, trade-off'ы и PHP/Laravel/Symfony-заметки.

## Как отвечать на интервью

- Начинайте с модели угроз: какие активы защищаем, кто атакующий, где trust boundary.
- Разделяйте prevention, detection и response: защита, мониторинг, план реакции.
- Говорите не только про фреймворк, а про протоколы, браузерную модель, инфраструктуру и операционные процессы.
- Для senior-ответа всегда упоминайте компромиссы: UX vs безопасность, stateless vs revocation, performance vs defense-in-depth.
- Избегайте ответа "используем Laravel/Symfony, значит безопасно". Фреймворк снижает риск, но не закрывает неверную бизнес-логику, секреты, SSRF, supply chain и неправильную конфигурацию.

## OWASP Top 10

OWASP Top 10 - не чеклист полного аудита, а обзор наиболее критичных классов рисков веб-приложений.

### A01 Broken Access Control

- Пользователь получает доступ к чужим данным, endpoint'ам, административным функциям или объектам.
- Частые причины: проверка только на UI, IDOR, отсутствие object-level authorization, неверные роли, mass assignment.
- PHP/Laravel: используйте Policies/Gates, `authorize()`, FormRequest authorization, `$fillable`/DTO вместо прямого `Model::create($request->all())`.
- Symfony: Voters, `#[IsGranted]`, access control в `security.yaml`, проверки на уровне use case/service.

Senior short answer: "Авторизация должна быть server-side, object-level и проверяться рядом с бизнес-операцией, а не только на route или UI".

### A02 Cryptographic Failures

- Слабое хранение паролей, отсутствие TLS, неправильное шифрование, хранение PII в логах, слабые ключи.
- Используйте password hashing для паролей, TLS для транспорта, envelope encryption/KMS для секретных данных.
- Не пишите собственную криптографию.

### A03 Injection

- SQL/NoSQL/OS/LDAP/template injection возникают, когда данные становятся кодом.
- Базовая защита: параметризованные запросы, ORM/query builder, allowlist, экранирование под конкретный контекст.
- Для shell: избегать shell-команд; если нужно - `escapeshellarg()`, allowlist команд и аргументов, изолированный worker.

### A04 Insecure Design

- Архитектурные ошибки: нет rate limiting, нет threat modeling, небезопасный recovery flow, невозможность отозвать токены.
- Лечится проектированием: abuse cases, secure-by-default flows, security requirements, review критичных сценариев.

### A05 Security Misconfiguration

- Debug включен в production, лишние headers отсутствуют, CORS `*` с credentials, открытые buckets, дефолтные пароли.
- PHP: `display_errors=Off`, `APP_DEBUG=false`, правильные permissions, закрытые `.env`, отключенный directory listing.

### A06 Vulnerable and Outdated Components

- Уязвимости Composer/npm/Docker images/OS packages.
- Нужны SCA, lock-файлы, регулярные обновления, SBOM, review transitive dependencies.

### A07 Identification and Authentication Failures

- Слабые пароли, credential stuffing, session fixation, неправильный reset password, отсутствие MFA.
- Меры: MFA для чувствительных ролей, rate limit, secure session rotation, безопасный password reset, мониторинг login anomalies.

### A08 Software and Data Integrity Failures

- Supply-chain атаки, небезопасные CI/CD, неподписанные артефакты, deserialization.
- Меры: pin versions, protected branches, least privilege tokens, signed releases/images, запрет unsafe deserialization.

### A09 Security Logging and Monitoring Failures

- Нет audit trail, события безопасности не логируются, невозможно расследовать инцидент.
- Логируйте auth events, permission denials, suspicious rate limit hits, token revocation, admin actions.
- Не логируйте passwords, tokens, Authorization headers, session IDs, full PII.

### A10 SSRF

- Сервер делает запрос на URL, контролируемый пользователем, и атакующий получает доступ к внутренней сети или metadata endpoint.
- Меры: allowlist host'ов, запрет private/link-local IP, DNS rebinding protection, egress firewall, timeout/size limit, запрет redirects или повторная проверка redirect target.

## SQL Injection

SQL injection - подмена структуры SQL-запроса через пользовательский ввод.

```php
// Плохо
$pdo->query("SELECT * FROM users WHERE email = '$email'");

// Хорошо
$stmt = $pdo->prepare('SELECT * FROM users WHERE email = :email');
$stmt->execute(['email' => $email]);
```

Laravel/Symfony notes:

- Eloquent/Query Builder и Doctrine DBAL безопасны при bindings, но `DB::raw()`, `whereRaw()`, DQL concatenation и dynamic order clauses опасны.
- Нельзя параметризовать имена колонок и направлений сортировки как values. Для них нужен allowlist.
- Least privilege DB user снижает blast radius, но не заменяет parameter binding.

Common pitfalls:

- "ORM защищает всегда" - нет, raw fragments и динамический SQL остаются риском.
- LIKE-запросы тоже требуют параметров; wildcard logic должна быть управляемой.
- Вторичная injection возможна, когда вредная строка сначала сохраняется, а позже используется в SQL.

Senior short answer: "Главная защита - prepared statements и allowlist для SQL identifiers; escaping вручную считаю запасным и контекстно-зависимым вариантом".

## XSS

XSS - выполнение JavaScript в браузере пользователя из-за небезопасного вывода данных.

Типы:

- Stored XSS: payload хранится в БД и показывается другим пользователям.
- Reflected XSS: payload приходит в request и сразу отражается в response.
- DOM XSS: клиентский JS небезопасно вставляет данные в DOM.

Защита:

- Context-aware escaping: HTML, attribute, URL, JS string - разные контексты.
- Blade `{{ $value }}` экранирует, `{!! $html !!}` опасен.
- Twig `{{ value }}` autoescape включен по умолчанию, `|raw` опасен.
- Для rich text используйте HTML sanitizer с allowlist тегов/атрибутов.
- CSP снижает ущерб, но не заменяет escaping.

Pitfalls:

- `htmlspecialchars()` не защищает внутри `<script>` без правильного JS escaping.
- Markdown/HTML preview часто становится Stored XSS.
- JSON внутри HTML должен кодироваться безопасно: в Laravel `@json`, `Js::from()`.

## CSRF

CSRF - браузер жертвы автоматически отправляет cookies на доверенный сайт, а атакующий инициирует нежелательное действие.

Защита:

- CSRF token для state-changing запросов.
- `SameSite=Lax/Strict` cookies.
- Проверка `Origin`/`Referer` как дополнительная мера.
- Не менять состояние через GET.

PHP/Laravel/Symfony notes:

- Laravel `VerifyCsrfToken` и Blade `@csrf`.
- Symfony CSRF Token Manager и form CSRF protection.
- Для stateless API с Bearer token в `Authorization` CSRF обычно не применим, потому что браузер не добавляет этот header автоматически.

Trade-off: cookies удобны и безопасны при правильных флагах, но требуют CSRF-защиты; Bearer tokens в browser storage проще для SPA, но повышают ущерб от XSS.

## SSRF

SSRF часто встречается в webhooks, URL preview, image import, PDF generation, integrations.

Checklist:

- Разрешать только нужные схемы: обычно `https`.
- Allowlist доменов, если бизнес-сценарий позволяет.
- Резолвить DNS и блокировать private ranges: `127.0.0.0/8`, `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`, `169.254.0.0/16`, IPv6 local ranges.
- Проверять IP после каждого redirect.
- Ограничить timeout, response size, методы и headers.
- Egress firewall на уровне сети надежнее application-only проверок.

Senior short answer: "Защита от SSRF должна быть многоуровневой: validation в коде плюс network egress policy, потому что DNS rebinding и redirects обходят простые проверки строк".

## Authentication vs Authorization

- Authentication: кто пользователь? Пример: login, MFA, session, token validation.
- Authorization: что ему разрешено? Пример: может ли редактировать конкретный invoice.

Типовая ошибка: проверили, что пользователь залогинен, но не проверили право на конкретный объект.

Senior short answer: "Authn подтверждает identity, authz принимает решение о доступе к действию и ресурсу; эти проверки независимы".

## Sessions и Cookies

Безопасные cookie-флаги:

- `HttpOnly`: JS не читает cookie, снижает ущерб от XSS.
- `Secure`: cookie только по HTTPS.
- `SameSite=Lax/Strict/None`: контроль cross-site отправки.
- `Path`, `Domain`: ограничивают область действия.

Session best practices:

- Rotate session ID после login и privilege change.
- Invalidate session на logout/password change.
- Idle timeout и absolute timeout для чувствительных систем.
- Хранить server-side session в Redis/DB, а в cookie только идентификатор.
- Не хранить секретные данные в cookie без шифрования и подписи.

Laravel/Symfony notes:

- Laravel шифрует и подписывает cookies; проверьте `SESSION_SECURE_COOKIE`, `SESSION_SAME_SITE`, `APP_KEY`.
- Symfony session cookie flags задаются в framework/session config; security firewall управляет login/logout.

## Password Hashing

Правильно:

- Использовать `password_hash()`/`password_verify()` или фреймворк-обертки.
- Алгоритмы: Argon2id предпочтителен, bcrypt все еще приемлем.
- Уникальная salt встроена в современные алгоритмы.
- Pepper возможен, но хранится отдельно в secrets manager, не в БД.
- Rehash при изменении параметров: `password_needs_rehash()`.

Неправильно:

- MD5/SHA1/SHA256 для паролей.
- Самодельная salt-конкатенация вместо password API.
- Логирование пароля или password reset token.

PHP example:

```php
$hash = password_hash($password, PASSWORD_ARGON2ID);

if (! password_verify($password, $hash)) {
    throw new RuntimeException('Invalid credentials');
}
```

Laravel: `Hash::make()`, `Hash::check()`, `Hash::needsRehash()`.

Symfony: Password Hasher component, `UserPasswordHasherInterface`.

## OAuth2 и OIDC Basics

OAuth2 - framework делегированной авторизации. Он отвечает на вопрос: "может ли client получить доступ к resource server от имени resource owner?"

OIDC - identity layer поверх OAuth2. Он добавляет authentication и `id_token`.

Роли OAuth2:

- Resource Owner: пользователь.
- Client: приложение, запрашивающее доступ.
- Authorization Server: выдает токены.
- Resource Server: API, принимающее access token.

Основные flows:

- Authorization Code + PKCE: основной flow для web/mobile/SPA.
- Client Credentials: machine-to-machine.
- Device Code: устройства без удобного ввода.
- Implicit и password grant считаются legacy и обычно не рекомендуются.

Tokens:

- Access token: короткоживущий токен доступа к API.
- Refresh token: долгоживущий токен для получения новых access tokens.
- ID token: OIDC-токен с утверждениями об authentication event; не предназначен как API access token.

Pitfalls:

- Путать OAuth2 с login. Для login используйте OIDC.
- Принимать `id_token` вместо access token на API.
- Не проверять `iss`, `aud`, `exp`, signature, nonce/state.
- Не использовать PKCE для public clients.

Senior short answer: "OAuth2 сам по себе не протокол аутентификации; для login нужен OIDC, а для современных user-facing clients - Authorization Code with PKCE".

## JWT: Best Practices и Pitfalls

JWT - формат токена с claims и подписью. Он не обязан быть stateless access token, но часто так используется.

Проверять обязательно:

- Signature и допустимый алгоритм.
- `iss` - ожидаемый issuer.
- `aud` - ожидаемый audience.
- `exp`, `nbf`, `iat` с разумным clock skew.
- `kid` только для выбора ключа из доверенного JWKS, не для произвольной загрузки ключа.

Pitfalls:

- Принимать `alg=none` или доверять алгоритму из header без allowlist.
- Использовать один secret для разных окружений/сервисов.
- Хранить JWT в `localStorage` в браузере и считать это безопасным от XSS.
- Класть PII/secrets в payload: JWT обычно только base64url encoded, не encrypted.
- Длинный TTL без механизма revocation.
- Не учитывать key rotation и `kid`.

Trade-offs:

- Stateless JWT хорошо масштабируется, но сложнее отзывать.
- Opaque token с introspection проще контролировать и отзывать, но требует обращения к auth server/cache.
- JWT удобен для межсервисной проверки, но увеличивает риск утечки claims и проблем с invalidation.

Senior short answer: "JWT безопасен не из-за формата, а из-за строгой валидации подписи, claims, TTL, key rotation и минимального payload".

## Access и Refresh Tokens

Access token:

- Короткий TTL: минуты, не дни.
- Минимальные scopes/claims.
- Предъявляется resource server'у.

Refresh token:

- Длиннее живет и требует более строгого хранения.
- Должен быть rotatable: refresh token rotation с reuse detection.
- Можно привязывать к device/session/client.
- Отзывается при logout, password change, suspicious activity.

Browser storage trade-off:

- HttpOnly Secure SameSite cookie защищает от чтения JS, но требует CSRF-модель.
- In-memory token снижает persistence, но теряется при reload.
- `localStorage` переживает reload, но уязвим к XSS.

## RBAC и ABAC

RBAC:

- Доступ через роли: admin, manager, editor.
- Просто объяснять и администрировать.
- Плохо масштабируется при множестве исключений и контекстных правил.

ABAC:

- Доступ через атрибуты пользователя, ресурса, действия и контекста.
- Пример: owner может редактировать draft invoice в своей organization до закрытия периода.
- Гибче, но сложнее тестировать и объяснять бизнесу.

Практика:

- Часто используют гибрид: coarse-grained RBAC + object/context-level ABAC.
- В Laravel это естественно ложится на Policies.
- В Symfony - на Voters.

Senior short answer: "Роли удобны для coarse-grained access, но реальные B2B-системы почти всегда требуют object-level policies с учетом tenant, owner, status и action".

## CORS

CORS - браузерный механизм, который разрешает frontend с одного origin читать ответы другого origin. Это не server-to-server firewall.

Основные правила:

- `Access-Control-Allow-Origin` должен быть конкретным origin для credentialed requests.
- Нельзя безопасно сочетать `Access-Control-Allow-Origin: *` и credentials.
- Preflight проверяет методы и headers.
- CORS не заменяет authentication/authorization.

Pitfalls:

- Динамически отражать любой `Origin` без allowlist.
- Считать, что CORS защищает API от curl/Postman/server-side requests.
- Разрешать credentials для всех subdomains без необходимости.

## Rate Limiting и Abuse Protection

Где применять:

- Login, password reset, registration, OTP/MFA, token refresh.
- Search/export/heavy endpoints.
- Public APIs и webhooks.

Ключи лимитов:

- IP, user ID, account ID, client ID, device fingerprint, route/action.
- Для login часто комбинируют IP + username/account.

Алгоритмы:

- Fixed window прост, но дает burst на границе окна.
- Sliding window точнее.
- Token bucket хорошо контролирует burst.

Laravel: `RateLimiter`, throttle middleware.

Symfony: RateLimiter component.

Senior short answer: "Rate limit должен учитывать abuse case: только IP недостаточно при botnet, только user ID недостаточно до login".

## File Upload Security

Риски:

- RCE через исполняемый файл.
- Stored XSS через SVG/HTML.
- Malware distribution.
- Zip bomb и decompression bomb.
- Path traversal и overwrite.
- EXIF/metadata leakage.

Checklist:

- Allowlist расширений и MIME, но не доверять только client-provided MIME.
- Проверять magic bytes/содержимое.
- Генерировать server-side имя файла.
- Хранить вне web root или отдавать через controlled endpoint/CDN.
- Запретить выполнение в upload-директории.
- Ограничить размер, количество, dimensions для изображений.
- Сканировать антивирусом для high-risk доменов.
- Нормализовать изображения через безопасную библиотеку, удалять metadata.

Laravel: validation rules `file`, `mimes`, `mimetypes`, `max`, storage disks.

Symfony: Validator constraints `File`, `Image`, безопасное имя через `SluggerInterface` + random suffix.

## Secrets Management

Secrets: DB passwords, API keys, JWT signing keys, OAuth client secrets, encryption keys, webhook secrets.

Правила:

- Не коммитить secrets в git.
- Разделять secrets по окружениям и сервисам.
- Least privilege для credentials.
- Rotation и revocation должны быть процедурой, а не героизмом.
- Использовать Vault/KMS/cloud secret managers/CI secret storage.
- Не передавать secrets в logs, metrics, traces, exceptions, frontend bundle.

PHP/Laravel/Symfony notes:

- `.env` удобен для local dev, но production лучше подключать через environment/secret manager.
- Laravel `APP_KEY` критичен для cookies/encryption; его утечка компрометирует данные.
- Symfony Secrets Vault помогает хранить encrypted secrets в проекте, но ключ расшифровки должен быть защищен.

Pitfalls:

- `phpinfo()` в production раскрывает environment.
- Secrets в Docker image layers.
- Secrets в CI logs из-за `set -x` или verbose команд.
- Один shared JWT secret для всех клиентов и окружений.

## Dependency и Supply-Chain Security: Composer/npm

Composer:

- Используйте `composer.lock` для приложений.
- Запускайте `composer audit` в CI.
- Ограничивайте plugins: `allow-plugins`.
- Проверяйте abandoned packages.
- Осторожно с scripts в `composer.json`.

npm:

- Используйте lockfile: `package-lock.json`, `pnpm-lock.yaml` или `yarn.lock`.
- `npm audit`/SCA в CI, но фильтруйте noise по exploitability.
- Осторожно с lifecycle scripts, typosquatting, dependency confusion.
- Pin registry/scope для private packages.

CI/CD:

- Protected branches, mandatory reviews, minimal CI token permissions.
- Не запускать untrusted PR с доступом к production secrets.
- Dependabot/Renovate + тесты + staged rollout.

Senior short answer: "Supply-chain security - это не только audit command, а контроль источника пакетов, lockfile, CI permissions, review transitive risk и быстрый patch process".

## Logging Sensitive Data

Что не логировать:

- Passwords, password reset tokens, refresh/access tokens, session IDs.
- Authorization/Cookie headers.
- Full card data, secrets, private keys.
- PII без необходимости.

Что логировать:

- Security events: login success/failure, MFA changes, password reset requested/completed, permission denied, admin actions.
- Correlation ID/request ID.
- User ID/account ID при законном основании и с учетом privacy.

Практика:

- Redaction middleware/processors для Monolog.
- Маскирование headers и request body.
- Разные уровни доступа к audit/security logs.
- Retention policy и tamper resistance для audit logs.

## Secure Headers

Базовые headers:

- `Content-Security-Policy`: снижает риск XSS и data injection.
- `Strict-Transport-Security`: принудительный HTTPS после первого доверенного визита.
- `X-Content-Type-Options: nosniff`: запрет MIME sniffing.
- `X-Frame-Options: DENY/SAMEORIGIN` или CSP `frame-ancestors`: clickjacking protection.
- `Referrer-Policy`: ограничение утечки URL/referrer.
- `Permissions-Policy`: ограничение browser capabilities.

Pitfalls:

- CSP в режиме `unsafe-inline` почти теряет смысл для XSS.
- HSTS preload требует уверенности, что все subdomains готовы к HTTPS.
- Headers не исправляют broken authz или SQL injection.

## TLS Basics

TLS защищает transport: confidentiality, integrity, server authentication.

Важно:

- Использовать TLS 1.2+ или 1.3.
- Правильная certificate chain и hostname validation.
- HSTS для web.
- Не отключать certificate verification в HTTP clients.
- mTLS полезен для service-to-service или партнерских интеграций, но усложняет lifecycle сертификатов.

Pitfalls:

- `verify => false` в Guzzle/Symfony HttpClient.
- Доверие к self-signed cert без pinning/CA management.
- Смешанный content на frontend.

## PHP/Laravel/Symfony Mini-Checklist

- `APP_DEBUG=false`/Symfony debug off в production.
- `display_errors=Off`, ошибки уходят в logs.
- Cookies: `Secure`, `HttpOnly`, `SameSite`.
- CSRF включен для cookie-based web forms.
- Policies/Voters покрывают object-level authorization.
- Passwords через `Hash`/Password Hasher/password API.
- Raw SQL только с bindings и allowlist identifiers.
- Uploads вне web root или без execute permissions.
- CORS allowlist, не wildcard для credentials.
- Rate limits на auth и expensive endpoints.
- Secrets не в git/image/logs.
- Dependency audit в CI.
- Security headers настроены и проверены.
- HTTP clients не ходят по user-provided URL без SSRF-защиты.

## Практические Interview Scenarios

### Как защитить endpoint `GET /invoices/{id}`?

Ответ:

- Authenticate user.
- Load invoice by ID.
- Authorize object-level access: tenant, ownership, role, status.
- Не полагаться на скрытие ID или UUID.
- Логировать denied access как security event с rate limiting при подозрительных переборах.

### Что выбрать: JWT или server-side session?

Ответ:

- Для классического web-приложения cookie + server-side session часто проще и безопаснее для revocation.
- Для API/microservices JWT удобен, если нужен decentralized verification.
- Если важен немедленный logout/revocation, нужен denylist/introspection/короткий TTL.
- В браузере хранение токенов требует отдельной XSS/CSRF модели.

### Как делать refresh token rotation?

Ответ:

- Каждый refresh request возвращает новый refresh token и инвалидирует старый.
- Повторное использование старого token - сигнал компрометации; инвалидируем token family/session.
- Храним hash refresh token в БД, не plaintext.
- Привязываем к client/device/session metadata и ограничиваем TTL.

### Как безопасно принимать webhook?

Ответ:

- Проверять подпись webhook body shared secret'ом или asymmetric signature.
- Проверять timestamp и replay window.
- Использовать idempotency key/event ID.
- Не доверять IP allowlist как единственной защите.
- Логировать verification failures без тела с секретными данными.

## Частые ошибки кандидатов

- "JWT нельзя украсть, он подписан" - подпись не шифрует и не защищает от replay при краже.
- "CORS защищает backend" - CORS ограничивает браузерное чтение response, не запросы вообще.
- "HTTPS достаточно" - TLS защищает транспорт, но не broken access control, XSS, SQLi.
- "UUID решает IDOR" - UUID снижает перебор, но не заменяет authorization.
- "Hashing и encryption одно и то же" - hash односторонний, encryption обратим при наличии ключа.
- "Sanitize input" как универсальный ответ - обычно важнее validate input и escape output по контексту.

## Self-Check Questions

- Чем authentication отличается от authorization?
- Почему `id_token` нельзя использовать как access token для API?
- Какие claims JWT вы проверите на resource server?
- Почему `localStorage` опасен для access/refresh tokens?
- Как защититься от IDOR в Laravel/Symfony?
- Почему CORS не является заменой авторизации?
- Как проверить user-provided URL на SSRF?
- Чем Argon2id лучше обычного SHA256 для паролей?
- Какие события безопасности стоит логировать?
- Как бы вы построили процесс rotation для leaked API key?
- Какие риски у `DB::raw()`/DQL string concatenation?
- Как безопасно обрабатывать SVG upload?
- Что дает HSTS и когда он может навредить?
- Как не утянуть production secrets в CI для untrusted PR?

## Mini-Practice

### Security Review Endpoint

Возьмите любой endpoint изменения состояния и проверьте:

- Есть ли authentication?
- Есть ли authorization на конкретный объект?
- Есть ли CSRF или Bearer-token модель?
- Есть ли rate limit/idempotency?
- Валидируются ли входные данные?
- Не логируются ли secrets/PII?
- Есть ли audit event?

### JWT Validation Checklist

- Signature verified.
- Algorithm allowlist enforced.
- `iss` matches expected issuer.
- `aud` matches this API.
- `exp` and `nbf` checked.
- Key rotation/JWKS cache handled.
- Token type/scope/permissions checked.
- Payload has no secrets/large PII.

### Secrets Incident Checklist

- Identify leaked secret and scope.
- Revoke/rotate immediately.
- Search logs, CI, images, git history.
- Deploy new secret through secret manager.
- Check suspicious usage.
- Document incident and add prevention control.

## Ссылки

- OWASP Top 10: https://owasp.org/www-project-top-ten/
- OWASP Cheat Sheet Series: https://cheatsheetseries.owasp.org/
- OWASP SQL Injection Prevention Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html
- OWASP Cross Site Scripting Prevention Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html
- OWASP CSRF Prevention Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html
- OWASP SSRF Prevention Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Server_Side_Request_Forgery_Prevention_Cheat_Sheet.html
- OAuth 2.0 RFC 6749: https://www.rfc-editor.org/rfc/rfc6749
- OAuth 2.0 Security Best Current Practice: https://datatracker.ietf.org/doc/html/draft-ietf-oauth-security-topics
- OpenID Connect Core: https://openid.net/specs/openid-connect-core-1_0.html
- JWT RFC 7519: https://www.rfc-editor.org/rfc/rfc7519
- JWT Best Current Practices RFC 8725: https://www.rfc-editor.org/rfc/rfc8725
- PHP password hashing docs: https://www.php.net/manual/en/book.password.php
- Laravel Security: https://laravel.com/docs/security
- Laravel Authentication: https://laravel.com/docs/authentication
- Laravel Authorization: https://laravel.com/docs/authorization
- Symfony Security: https://symfony.com/doc/current/security.html
- Symfony Password Hasher: https://symfony.com/doc/current/security/passwords.html
- Symfony CSRF: https://symfony.com/doc/current/security/csrf.html
- Composer audit: https://getcomposer.org/doc/03-cli.md#audit
- npm audit: https://docs.npmjs.com/cli/commands/npm-audit
