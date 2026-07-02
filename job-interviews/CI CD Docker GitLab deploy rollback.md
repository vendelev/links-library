# CI/CD, Docker, GitLab, deploy, rollback

Практический конспект для Lead/Senior PHP backend developer перед собеседованием.

## CI/CD: суть и принципы

CI/CD - это дисциплина быстрой и безопасной доставки изменений.

- **CI, Continuous Integration**: каждое изменение регулярно сливается в основную ветку и автоматически проверяется тестами, статическим анализом, сборкой.
- **CD, Continuous Delivery**: артефакт всегда готов к выкладке, но production deploy может требовать ручного подтверждения.
- **CD, Continuous Deployment**: успешное изменение автоматически попадает в production без ручного шага.

Ключевые принципы:

- pipeline должен быть воспроизводимым, быстрым и детерминированным;
- один и тот же артефакт должен проходить путь от build до production;
- конфигурация отделена от кода;
- секреты не хранятся в репозитории;
- deploy должен быть наблюдаемым и обратимым;
- rollback должен быть заранее спроектирован, а не придуман во время инцидента.

Senior-level short answer:

> Хороший CI/CD не просто запускает тесты. Он строит доверенный артефакт, проверяет его качество и безопасность, доставляет его через окружения с контролируемым риском, дает быстрый rollback и оставляет трассируемый audit trail: кто, что, когда и почему выкатил.

## GitLab CI: stages, jobs, runners, artifacts, cache

GitLab CI описывается в `.gitlab-ci.yml`.

Основные сущности:

- **pipeline**: полный запуск CI/CD для commit, merge request, tag или schedule;
- **stage**: логическая фаза, например `lint`, `test`, `build`, `deploy`;
- **job**: конкретная задача внутри stage;
- **runner**: агент, который выполняет job;
- **artifact**: результат job, который нужен дальше или для скачивания;
- **cache**: переиспользуемые зависимости между pipeline/job для ускорения.

Пример стадий:

```yaml
stages:
  - lint
  - test
  - build
  - security
  - deploy
```

Artifacts vs cache:

- **artifacts** нужны для передачи результата: coverage, junit report, собранный build, dotenv-файл, docker image metadata;
- **cache** нужен для ускорения: Composer cache, npm cache, промежуточные директории;
- artifacts привязаны к pipeline/job и имеют срок хранения;
- cache может переиспользоваться между pipeline и не должен быть источником истины.

Типичные ошибки:

- использовать cache как обязательный build output;
- не пинить версии docker images;
- запускать deploy из merge request pipeline;
- хранить `.env`, SSH keys или registry tokens в репозитории;
- не разделять protected/non-protected variables;
- использовать один shared runner для production deploy без контроля доступа.

Senior-level short answer:

> Stages задают порядок, jobs делают работу, runners исполняют jobs. Artifacts передают проверенный результат между стадиями, cache только ускоряет повторные запуски и не должен влиять на корректность.

## Типовой PHP pipeline

Минимальный production-oriented pipeline:

1. Validate Composer и lock-файл.
2. Установить зависимости через `composer install`.
3. Запустить PHP lint.
4. Запустить статический анализ: PHPStan/Psalm.
5. Запустить code style: PHP CS Fixer/Pint/ECS.
6. Запустить unit/integration tests.
7. Собрать Docker image.
8. Прогнать security scans.
9. Push image в registry.
10. Deploy на нужное окружение.
11. Smoke tests и health checks после deploy.

Пример jobs на уровне идеи:

```yaml
composer:
  stage: test
  image: composer:2
  script:
    - composer validate --strict
    - composer install --no-interaction --prefer-dist --no-progress

phpstan:
  stage: test
  image: php:8.3-cli
  script:
    - vendor/bin/phpstan analyse

tests:
  stage: test
  image: php:8.3-cli
  script:
    - vendor/bin/phpunit --log-junit junit.xml
  artifacts:
    reports:
      junit: junit.xml
```

Практический tradeoff:

- быстрые проверки запускают на каждый MR;
- долгие end-to-end или security scans можно запускать nightly или перед release;
- production deploy должен идти только из protected branch/tag;
- deploy job часто делают `manual`, если процесс Continuous Delivery, а не Continuous Deployment.

## Composer install в CI и production

Для CI:

```bash
composer install --no-interaction --prefer-dist --no-progress
```

Для production image/deploy:

```bash
composer install --no-dev --no-interaction --prefer-dist --optimize-autoloader --classmap-authoritative
```

Важно:

- commit `composer.lock` для приложений обязателен;
- `composer update` не должен выполняться при deploy;
- `composer install` должен падать, если lock не соответствует `composer.json`;
- credentials для private packages передаются через CI variables или Composer auth config, но не коммитятся;
- кешируйте Composer download cache, а не `vendor` как источник истины.

Common pitfall:

> `composer update` в pipeline делает build недетерминированным: сегодня и завтра один и тот же commit может собрать разные зависимости.

## Dockerfile best practices для PHP

Хороший Dockerfile для PHP должен быть маленьким, воспроизводимым и безопасным.

Практики:

- пинить базовый image: `php:8.3.8-fpm-alpine` или digest;
- использовать multi-stage build;
- сначала копировать `composer.json` и `composer.lock`, затем ставить зависимости;
- не запускать приложение от root;
- не класть secrets в image;
- отделять build dependencies от runtime dependencies;
- включать production настройки OPcache;
- добавлять `HEALTHCHECK`, если это уместно для платформы;
- не копировать `.git`, tests, local config, cache в production image;
- использовать `.dockerignore`.

Пример multi-stage:

```dockerfile
FROM composer:2 AS vendor
WORKDIR /app
COPY composer.json composer.lock ./
RUN composer install \
    --no-dev \
    --no-interaction \
    --prefer-dist \
    --optimize-autoloader \
    --classmap-authoritative

FROM php:8.3-fpm-alpine AS runtime
WORKDIR /app
RUN addgroup -g 1000 app && adduser -D -G app -u 1000 app
COPY --from=vendor /app/vendor ./vendor
COPY . .
USER app
```

Tradeoff Alpine vs Debian:

- Alpine обычно меньше;
- Debian-based images часто проще для PHP extensions, ICU, GD, system libraries;
- для production важнее предсказуемость и поддерживаемость, чем минимальный размер любой ценой.

Senior-level short answer:

> Docker image должен содержать только runtime, код и проверенные зависимости. Build должен быть детерминированным, без секретов внутри слоев, с non-root пользователем и минимальной attack surface.

## Docker Compose для dev environment

`docker compose` удобен для локальной разработки и integration tests.

Типичный стек:

- `php-fpm` или `php-cli`;
- `nginx`;
- `postgres`/`mysql`;
- `redis`;
- `rabbitmq`;
- `mailpit`;
- optional: `adminer`, `minio`, `clickhouse`.

Практики:

- не использовать production secrets локально;
- хранить `.env.example`, но не `.env`;
- volume для исходников удобен в dev, но не нужен в production;
- healthchecks помогают корректно стартовать dependent services;
- dev compose не должен становиться production orchestrator.

Common pitfall:

> `depends_on` не означает, что база готова принимать подключения. Нужны healthchecks или retry logic в приложении/миграциях.

## Secrets и env

12-factor подход: config хранится в environment, а не в коде.

Что важно:

- secrets в GitLab CI Variables, Vault, Kubernetes Secrets, AWS/GCP/Azure secret managers;
- production variables должны быть protected и masked;
- не печатать secrets в logs;
- не передавать secrets через Docker build args, если они попадут в image layers;
- разные окружения имеют разные credentials;
- rotation secrets должна быть возможна без пересборки image.

Senior-level short answer:

> Секреты не должны попадать ни в git, ни в docker layers, ни в artifacts, ни в logs. Runtime получает их из защищенного secret store или protected CI variables.

## Deploy: общая модель

Надежный deploy обычно состоит из шагов:

1. Выбрать проверенный image/artifact.
2. Применить инфраструктурную конфигурацию.
3. Выполнить backward-compatible миграции.
4. Запустить новую версию рядом со старой или постепенно заменить старую.
5. Выполнить readiness checks.
6. Переключить трафик.
7. Запустить smoke tests.
8. Мониторить golden signals и business metrics.

Важная идея:

> Deploy кода и release функции - разные вещи. Feature flags позволяют выкатить код без немедленного включения поведения для всех пользователей.

## Database migrations during deploy

Миграции - частая причина сложных rollback.

Безопасный подход: expand/contract.

1. **Expand**: добавить новую колонку/таблицу/индекс без удаления старого поведения.
2. Выложить код, который умеет работать со старой и новой схемой.
3. Backfill данных отдельной задачей.
4. Переключить чтение/запись на новую схему.
5. **Contract**: удалить старую колонку/код в следующем релизе.

Правила:

- миграции должны быть идемпотентными насколько возможно;
- long-running migrations не должны блокировать таблицы в peak time;
- индексы на больших таблицах создавать online/concurrently, если DB это поддерживает;
- destructive migrations отделять от deploy кода;
- rollback кода не всегда откатывает схему;
- down-миграции часто опасны в production, если приводят к потере данных.

Senior-level short answer:

> Для zero-downtime deploy миграции должны быть backward-compatible. Сначала расширяем схему, потом выкатываем код, потом чистим старое. Rollback кода должен работать на новой схеме.

## Zero-downtime deploy

Zero-downtime deploy означает, что пользовательский трафик не получает ошибок из-за выкладки.

Требования:

- приложение stateless или состояние вынесено наружу;
- readiness/liveness probes;
- graceful shutdown;
- backward-compatible API и DB schema;
- прогрев cache/OPcache при необходимости;
- корректная работа очередей и cron jobs;
- контроль версий assets и static files;
- health checks до включения инстанса в балансировщик.

PHP-specific нюансы:

- PHP-FPM workers должны корректно завершать текущие запросы;
- OPcache может держать старый код, если deploy делается поверх файловой системы;
- при Docker/Kubernetes проще заменить immutable контейнер, чем обновлять файлы на месте;
- queue workers нужно перезапускать после deploy, чтобы они подхватили новый код.

Common pitfall:

> Код уже выкатили, HTTP трафик работает, но старые queue workers продолжают выполнять jobs старой версией и ломают формат данных.

## Blue-green, canary, rolling deploy

**Blue-green**:

- есть две production-like среды: blue и green;
- новая версия выкатывается на неактивную среду;
- после проверок трафик переключается;
- rollback быстрый: вернуть трафик назад.

Tradeoff: дороже по инфраструктуре, но очень быстрый rollback.

**Canary**:

- новая версия получает малую долю трафика;
- метрики сравниваются со stable версией;
- доля постепенно увеличивается.

Tradeoff: сложнее observability и routing, зато ниже blast radius.

**Rolling deploy**:

- инстансы заменяются постепенно;
- доступность сохраняется, если есть несколько реплик;
- стандартный подход в Kubernetes.

Tradeoff: некоторое время работают две версии одновременно, значит нужны совместимые API, events, DB schema.

Senior-level short answer:

> Blue-green оптимален для быстрого переключения и rollback, canary - для снижения риска через малый процент трафика, rolling - базовый экономичный способ постепенно заменить реплики. Во всех случаях важна совместимость версий.

## Feature environments

Feature environments или review apps - временные окружения на branch/MR.

Польза:

- QA и product могут проверить изменения до merge;
- проще тестировать интеграции;
- меньше конфликтов на общем staging;
- удобно для demo.

Что предусмотреть:

- auto-stop environment;
- ограничение ресурсов;
- seed/test data;
- отдельные credentials;
- запрет доступа к production data;
- cleanup после merge/close MR.

## Rollback strategy

Rollback - это не только `git revert`.

Виды rollback:

- переключить трафик на предыдущую версию;
- redeploy предыдущего Docker image по tag/digest;
- выключить feature flag;
- откатить config;
- остановить consumer/worker;
- применить forward-fix;
- компенсировать данные отдельным скриптом.

Правила хорошего rollback:

- предыдущий артефакт известен и доступен;
- release связан с commit SHA, tag, image digest и changelog;
- миграции совместимы назад хотя бы на одну версию;
- есть runbook: кто принимает решение, какие метрики смотрим, какие команды выполняем;
- rollback регулярно проверяется на staging/game day.

Когда rollback плохая идея:

- уже изменены данные в новом формате;
- external API получил необратимые side effects;
- старая версия несовместима с новой схемой БД;
- проще и безопаснее сделать forward-fix.

Senior-level short answer:

> Rollback должен быть заранее проверенным сценарием. Самый надежный rollback - переключить трафик или image назад, но только если миграции и данные совместимы. Для risky features лучше feature flags и canary.

## Release notes, tagging, versioning

Трассируемость релиза важна для аудита и диагностики.

Хороший release содержит:

- git tag или release branch;
- commit SHA;
- Docker image tag и digest;
- список merge requests;
- breaking changes;
- миграции;
- feature flags;
- инструкции rollback;
- ссылки на dashboards/logs;
- автора/approver релиза.

Tagging практики:

- semantic versioning для продуктов/пакетов, где это имеет смысл;
- immutable image tags для production: `app:1.12.3`, `app:git-sha`, digest;
- не использовать `latest` как production reference;
- release notes генерировать из MR labels/changelog, но проверять вручную для важных релизов.

## Observability during deploy

Без observability deploy превращается в надежду.

Что смотреть:

- error rate;
- latency p95/p99;
- traffic/request rate;
- saturation: CPU, memory, DB connections, queue lag;
- HTTP 5xx/4xx;
- failed jobs;
- logs новой версии;
- business metrics: checkout, registration, payment success rate;
- external integrations errors.

Практики:

- аннотировать deploy в Grafana/Datadog/New Relic/Sentry;
- логировать version/commit SHA в приложении;
- добавить `/health` и `/ready` endpoints;
- smoke tests после переключения трафика;
- alerting должен отличать canary от full rollout;
- correlation ID помогает связывать logs/traces.

Senior-level short answer:

> Я не считаю deploy завершенным после успешной команды deploy. Он завершен после health checks, smoke tests и короткого окна наблюдения по техническим и бизнес-метрикам.

## Security scans и dependency audit

Что проверять в pipeline:

- Composer dependencies: known vulnerabilities;
- Docker image vulnerabilities;
- secrets detection;
- SAST;
- dependency license policy;
- IaC/container config: privileged containers, root user, exposed ports;
- outdated base images;
- SBOM, если требуется compliance.

Инструменты:

- `composer audit`;
- GitLab Dependency Scanning;
- GitLab Container Scanning;
- GitLab Secret Detection;
- Trivy, Grype, Syft;
- OWASP Dependency-Check;
- Semgrep/PHPStan security rules.

Tradeoff:

- блокировать production на critical/high vulnerabilities полезно;
- блокировать каждый MR на medium без triage может парализовать разработку;
- нужен процесс exception/acceptance с owner и сроком пересмотра.

Senior-level short answer:

> Security scanning должен быть частью pipeline, но с понятной политикой severity, false positives и exception process. Иначе команда либо игнорирует отчеты, либо перестает доставлять изменения.

## Практические объяснения для интервью

Коротко: зачем нужен CI/CD?

> Чтобы каждое изменение проходило одинаковую автоматическую проверку и доставлялось предсказуемо. Это снижает человеческий фактор, ускоряет feedback loop и делает релизы маленькими, частыми и обратимыми.

Коротко: чем artifact отличается от Docker image?

> Artifact - общий термин для результата job. Docker image - конкретный runtime artifact, который можно запускать. В modern deploy часто главным артефактом является immutable Docker image.

Коротко: почему нельзя собирать заново на production?

> Потому что production должен запускать уже проверенный артефакт. Если собирать заново, можно получить другие зависимости, другой base image или ошибку окружения, которую CI не проверял.

Коротко: как выкатывать миграции без downtime?

> Делать backward-compatible изменения: сначала добавить новую схему, затем выкатить код, затем backfill, затем переключить чтение/запись, и только в следующем релизе удалить старую схему.

Коротко: что делать, если deploy сломал production?

> Сначала оценить blast radius по метрикам и логам, остановить rollout или отключить feature flag, если возможно. Если предыдущая версия совместима - откатить traffic/image. Если проблема в данных или миграции - выбрать forward-fix или компенсирующий скрипт по runbook.

## Common pitfalls and tradeoffs

- Слишком медленный pipeline снижает частоту feedback и провоцирует обход CI.
- Один огромный deploy в неделю рискованнее маленьких ежедневных релизов.
- `latest` в production делает rollback и аудит сложными.
- Миграции с `DROP COLUMN` в одном релизе с кодом часто ломают rollback.
- Общий staging быстро становится бутылочным горлышком.
- Feature flags без cleanup превращаются в технический долг.
- Canary без хороших метрик дает ложное чувство безопасности.
- Shared runners удобны, но для production deploy лучше controlled/protected runners.
- Секреты в Docker build args могут остаться в history/layers.
- Кеширование `vendor` может скрывать проблемы с lock-файлом.

## Mini-practice

Задание 1: спроектируй pipeline для Laravel/Symfony API.

Чеклист ответа:

- `composer validate`;
- `composer install` из lock;
- PHP lint;
- PHPStan/Psalm;
- code style;
- PHPUnit/integration tests;
- build Docker image;
- security scans;
- push image по immutable tag;
- deploy staging;
- smoke tests;
- manual production deploy;
- deploy annotation и мониторинг;
- rollback на предыдущий image.

Задание 2: опиши safe migration для переименования колонки.

Чеклист ответа:

- добавить новую колонку;
- писать в обе колонки или синхронизировать;
- backfill;
- переключить чтение на новую колонку;
- убедиться по метрикам;
- удалить старую колонку отдельным релизом.

Задание 3: подготовь rollback runbook.

Чеклист ответа:

- где найти предыдущий release/image digest;
- кто принимает решение rollback;
- какие метрики являются trigger;
- команда rollback;
- проверка health/smoke;
- коммуникация в incident channel;
- postmortem и follow-up tasks.

## Self-check questions

- Чем Continuous Delivery отличается от Continuous Deployment?
- Почему artifacts и cache нельзя использовать одинаково?
- Какие GitLab variables должны быть protected и masked?
- Почему production deploy лучше делать из tag или protected branch?
- Как ускорить Composer install без потери воспроизводимости?
- Какие проблемы возникают при rolling deploy двух несовместимых версий?
- Почему rollback миграций сложнее rollback кода?
- Как queue workers влияют на zero-downtime deploy PHP приложения?
- Что должно быть в release notes для production релиза?
- Какие метрики ты проверишь в первые 10 минут после deploy?
- Когда лучше canary, а когда blue-green?
- Как не допустить попадания secrets в Docker image?
- Что делать с false positives в security scans?

## Checklists

Перед merge:

- pipeline зеленый;
- тесты покрывают критичный сценарий;
- нет секретов в diff;
- миграции backward-compatible;
- feature flag добавлен для рискованного поведения;
- release notes обновлены, если изменение user-facing или operationally significant.

Перед production deploy:

- выбран конкретный image digest/tag;
- staging проверен;
- migrations оценены по длительности и lock risk;
- rollback plan понятен;
- dashboards и alerts готовы;
- on-call знает про релиз;
- external dependencies доступны.

После deploy:

- health checks прошли;
- smoke tests прошли;
- error rate не вырос;
- latency стабильна;
- queue lag нормальный;
- business metrics не просели;
- deploy annotation добавлена;
- incident channel закрыт или оставлен на наблюдение.

## Ссылки

- GitLab CI/CD documentation: https://docs.gitlab.com/ci/
- GitLab CI/CD YAML reference: https://docs.gitlab.com/ci/yaml/
- GitLab environments and deployments: https://docs.gitlab.com/ci/environments/
- GitLab security scanning: https://docs.gitlab.com/user/application_security/
- Dockerfile best practices: https://docs.docker.com/develop/develop-images/dockerfile_best-practices/
- Docker Compose documentation: https://docs.docker.com/compose/
- Docker multi-stage builds: https://docs.docker.com/build/building/multi-stage/
- Composer deployment recommendations: https://getcomposer.org/doc/03-cli.md#install-i
- Composer audit: https://getcomposer.org/doc/03-cli.md#audit
- Laravel deployment: https://laravel.com/docs/deployment
- Symfony deployment: https://symfony.com/doc/current/deployment.html
- The Twelve-Factor App: https://12factor.net/
- OWASP CI/CD Security Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/CI_CD_Security_Cheat_Sheet.html
