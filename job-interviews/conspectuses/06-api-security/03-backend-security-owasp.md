# Backend Security и OWASP

<!-- markdownlint-disable MD013 -->

Гайд для Lead/Senior PHP backend developer перед интервью. Фокус: модель угроз, OWASP Top 10, типовые backend-уязвимости, secure defaults и практики Laravel/Symfony.

## Как отвечать на интервью

- Начинайте с модели угроз: какие активы защищаем, кто атакующий, где trust boundary.
- Разделяйте prevention, detection и response: защита, мониторинг, план реакции.
- Говорите не только про фреймворк, а про протоколы, браузерную модель, инфраструктуру и операционные процессы.
- Для senior-ответа упоминайте компромиссы: UX vs безопасность, stateless vs revocation, performance vs defense-in-depth.
- Избегайте ответа "используем Laravel/Symfony, значит безопасно". Фреймворк снижает риск, но не закрывает неверную бизнес-логику, секреты, SSRF, supply chain и неправильную конфигурацию.

## OWASP Top 10

OWASP Top 10 - не чеклист полного аудита, а обзор наиболее критичных классов рисков веб-приложений.

### A01 Broken Access Control

- Пользователь получает доступ к чужим данным, endpoint'ам, административным функциям или объектам.
- Частые причины: проверка только на UI, IDOR, отсутствие object-level authorization, неверные роли, mass assignment.
- PHP/Laravel: используйте Policies/Gates, `authorize()`, FormRequest authorization, `$fillable`/DTO вместо прямого `Model::create($request->all())`.
- Symfony: Voters, `#[IsGranted]`, access control в `security.yaml`, проверки на уровне use case/service.

Senior short answer:

> Авторизация должна быть server-side, object-level и проверяться рядом с бизнес-операцией, а не только на route или UI.

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

Senior short answer:

> Главная защита - prepared statements и allowlist для SQL identifiers; escaping вручную считаю запасным и контекстно-зависимым вариантом.

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

Trade-off:

- Cookies удобны и безопасны при правильных флагах, но требуют CSRF-защиты.
- Bearer tokens в browser storage проще для SPA, но повышают ущерб от XSS.

## SSRF

SSRF часто встречается в webhooks, URL preview, image import, PDF generation и integrations.

Checklist:

- Разрешать только нужные схемы: обычно `https`.
- Allowlist доменов, если бизнес-сценарий позволяет.
- Резолвить DNS и блокировать private ranges: `127.0.0.0/8`, `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`, `169.254.0.0/16`, IPv6 local ranges.
- Проверять IP после каждого redirect.
- Ограничить timeout, response size, методы и headers.
- Egress firewall на уровне сети надежнее application-only проверок.

Senior short answer:

> Защита от SSRF должна быть многоуровневой: validation в коде плюс network egress policy, потому что DNS rebinding и redirects обходят простые проверки строк.

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

Senior short answer:

> Rate limit должен учитывать abuse case: только IP недостаточно при botnet, только user ID недостаточно до login.

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

## Практические interview scenarios

### Как защитить endpoint `GET /invoices/{id}`?

Ответ:

- Authenticate user.
- Load invoice by ID.
- Authorize object-level access: tenant, ownership, role, status.
- Не полагаться на скрытие ID или UUID.
- Логировать denied access как security event с rate limiting при подозрительных переборах.

### Как безопасно принимать webhook?

Ответ:

- Проверять подпись webhook body shared secret'ом или asymmetric signature.
- Проверять timestamp и replay window.
- Использовать idempotency key/event ID.
- Не доверять IP allowlist как единственной защите.
- Логировать verification failures без тела с секретными данными.

## Частые ошибки кандидатов

- "CORS защищает backend" - CORS ограничивает браузерное чтение response, не запросы вообще.
- "HTTPS достаточно" - TLS защищает транспорт, но не broken access control, XSS, SQLi.
- "UUID решает IDOR" - UUID снижает перебор, но не заменяет authorization.
- "Hashing и encryption одно и то же" - hash односторонний, encryption обратим при наличии ключа.
- "Sanitize input" как универсальный ответ - обычно важнее validate input и escape output по контексту.

## Mini-Practice: Security Review Endpoint

Возьмите любой endpoint изменения состояния и проверьте:

- Есть ли authentication?
- Есть ли authorization на конкретный объект?
- Есть ли CSRF или Bearer-token модель?
- Есть ли rate limit/idempotency?
- Валидируются ли входные данные?
- Не логируются ли secrets/PII?
- Есть ли audit event?

## Self-check questions

- Чем authentication отличается от authorization?
- Как защититься от IDOR в Laravel/Symfony?
- Почему CORS не является заменой авторизации?
- Как проверить user-provided URL на SSRF?
- Какие риски у `DB::raw()`/DQL string concatenation?
- Как безопасно обрабатывать SVG upload?
- Что дает HSTS и когда он может навредить?

## Дополнительное чтение

- OWASP Top 10 - https://owasp.org/www-project-top-ten/
- OWASP Cheat Sheet Series - https://cheatsheetseries.owasp.org/
- OWASP SQL Injection Prevention Cheat Sheet - https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html
- OWASP Cross Site Scripting Prevention Cheat Sheet - https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html
- OWASP CSRF Prevention Cheat Sheet - https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html
- OWASP SSRF Prevention Cheat Sheet - https://cheatsheetseries.owasp.org/cheatsheets/Server_Side_Request_Forgery_Prevention_Cheat_Sheet.html
- OWASP File Upload Cheat Sheet - https://cheatsheetseries.owasp.org/cheatsheets/File_Upload_Cheat_Sheet.html
- OWASP Content Security Policy Cheat Sheet - https://cheatsheetseries.owasp.org/cheatsheets/Content_Security_Policy_Cheat_Sheet.html
- OWASP Secure Headers Project - https://owasp.org/www-project-secure-headers/
- Mozilla Web Security Guidelines - https://infosec.mozilla.org/guidelines/web_security
- MDN CORS - https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CORS
- MDN Content Security Policy - https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP
- Laravel Security - https://laravel.com/docs/security
- Symfony Security - https://symfony.com/doc/current/security.html
- Symfony RateLimiter - https://symfony.com/doc/current/rate_limiter.html
