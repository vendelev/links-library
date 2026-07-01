# Auth, OAuth, JWT и Secrets

<!-- markdownlint-disable MD013 -->

Конспект про authentication, authorization, sessions, cookies, password hashing, OAuth2/OIDC, JWT, access/refresh tokens, RBAC/ABAC, secrets, sensitive logging и supply-chain security.

## Authentication vs Authorization

- Authentication: кто пользователь? Пример: login, MFA, session, token validation.
- Authorization: что ему разрешено? Пример: может ли редактировать конкретный invoice.
- Типовая ошибка: проверили, что пользователь залогинен, но не проверили право на конкретный объект.

Senior short answer:

> Authn подтверждает identity, authz принимает решение о доступе к действию и ресурсу; эти проверки независимы.

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

Senior short answer:

> OAuth2 сам по себе не протокол аутентификации; для login нужен OIDC, а для современных user-facing clients - Authorization Code with PKCE.

## JWT: best practices и pitfalls

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

Senior short answer:

> JWT безопасен не из-за формата, а из-за строгой валидации подписи, claims, TTL, key rotation и минимального payload.

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

Senior short answer:

> Роли удобны для coarse-grained access, но реальные B2B-системы почти всегда требуют object-level policies с учетом tenant, owner, status и action.

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

## Dependency и Supply-Chain Security

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

Senior short answer:

> Supply-chain security - это не только audit command, а контроль источника пакетов, lockfile, CI permissions, review transitive risk и быстрый patch process.

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

## Practical interview scenarios

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

### Как построить процесс rotation для leaked API key?

Ответ:

- Определить leaked secret и его scope.
- Revoke/rotate немедленно.
- Проверить logs, CI, images, git history.
- Доставить новый secret через secret manager.
- Проверить suspicious usage.
- Задокументировать incident и добавить prevention control.

## Частые ошибки кандидатов

- "JWT нельзя украсть, он подписан" - подпись не шифрует и не защищает от replay при краже.
- "OAuth2 это login" - OAuth2 про delegated authorization, login требует OIDC.
- "id_token можно слать в API" - API должен принимать access token, а `id_token` предназначен для client.
- "localStorage удобен, значит норм" - XSS превращает localStorage в источник кражи токенов.
- "Один JWT secret на все сервисы проще" - так растет blast radius и ломается нормальная rotation.

## JWT validation checklist

- Signature verified.
- Algorithm allowlist enforced.
- `iss` matches expected issuer.
- `aud` matches this API.
- `exp` and `nbf` checked.
- Key rotation/JWKS cache handled.
- Token type/scope/permissions checked.
- Payload has no secrets/large PII.

## Secrets incident checklist

- Identify leaked secret and scope.
- Revoke/rotate immediately.
- Search logs, CI, images, git history.
- Deploy new secret through secret manager.
- Check suspicious usage.
- Document incident and add prevention control.

## Self-check questions

- Почему `id_token` нельзя использовать как access token для API?
- Какие claims JWT вы проверите на resource server?
- Почему `localStorage` опасен для access/refresh tokens?
- Чем Argon2id лучше обычного SHA256 для паролей?
- Какие события безопасности стоит логировать?
- Как бы вы построили процесс rotation для leaked API key?
- Как не утянуть production secrets в CI для untrusted PR?

## Дополнительное чтение

- OAuth 2.0 RFC 6749 - https://www.rfc-editor.org/rfc/rfc6749
- OAuth 2.0 Security Best Current Practice - https://datatracker.ietf.org/doc/html/draft-ietf-oauth-security-topics
- OAuth 2.1 draft - https://datatracker.ietf.org/doc/html/draft-ietf-oauth-v2-1
- OpenID Connect Core - https://openid.net/specs/openid-connect-core-1_0.html
- JWT RFC 7519 - https://www.rfc-editor.org/rfc/rfc7519
- JWT Best Current Practices RFC 8725 - https://www.rfc-editor.org/rfc/rfc8725
- Auth0: ID Token vs Access Token - https://auth0.com/blog/id-token-access-token-what-is-the-difference/
- Auth0: Refresh Token Rotation - https://auth0.com/docs/secure/tokens/refresh-tokens/refresh-token-rotation
- OWASP Authentication Cheat Sheet - https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- OWASP Authorization Cheat Sheet - https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html
- OWASP Password Storage Cheat Sheet - https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html
- OWASP Secrets Management Cheat Sheet - https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
- OWASP Logging Cheat Sheet - https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html
- PHP password hashing docs - https://www.php.net/manual/en/book.password.php
- Laravel Authentication - https://laravel.com/docs/authentication
- Laravel Authorization - https://laravel.com/docs/authorization
- Symfony Security - https://symfony.com/doc/current/security.html
- Symfony Password Hasher - https://symfony.com/doc/current/security/passwords.html
- Symfony CSRF - https://symfony.com/doc/current/security/csrf.html
- Composer audit - https://getcomposer.org/doc/03-cli.md#audit
- npm audit - https://docs.npmjs.com/cli/commands/npm-audit
