# Coverage map: перенос старых production-конспектов

Карта нужна для проверки, что при перегруппировке не потерялись темы из исходных файлов. Старые файлы не удалялись; новый раздел `05-production` содержит переработанную и сгруппированную версию.

## Новая структура

- `00-index.md` - навигация и связи между файлами.
- `01-observability-sentry-opentelemetry.md` - telemetry, Sentry, OTel, SLO, alerts, dashboards, PHP integrations.
- `02-http-dns-tls-nginx.md` - путь HTTP-запроса, DNS/TCP/TLS, HTTP semantics, Nginx, PHP-FPM, status codes, 499/502/504.
- `03-linux-production-debugging.md` - Linux commands, CPU/memory/disk/network/fd/systemd/cron/strace/Docker/PHP-FPM diagnostics.
- `04-ci-cd-docker-gitlab-deploy-rollback.md` - CI/CD, GitLab, Dockerfile, secrets, deploy, migrations, rollout strategies, rollback.
- `05-object-storage-s3-minio-cdn-media.md` - S3/MinIO/CDN/media files, direct upload, lifecycle, security, PHP integration.
- `06-production-incident-playbook.md` - общий incident flow, triage, mitigation, postmortem.
- `07-timeouts-retries-idempotency-backpressure.md` - timeout budget, retries, idempotency, backpressure, long-running work.

## Source: Observability Sentry OpenTelemetry Logs Metrics Traces.md

| Старый раздел | Новый файл |
| --- | --- |
| Короткая карта темы | `01`, `00` |
| Observability vs Monitoring | `01` |
| Logs, Metrics, Traces | `01` |
| Структурированное логирование | `01`, `06` |
| Correlation ID и Trace ID | `01`, `06` |
| Distributed Tracing | `01` |
| OpenTelemetry | `01` |
| Context propagation | `01`, `07` |
| Sentry Errors/Performance/Releases | `01`, `04`, `06` |
| Metrics: RED и USE | `01`, `06` |
| SLI, SLO, SLA | `01`, `06` |
| Alerting | `01`, `06` |
| Dashboards | `01`, `06` |
| Incident Debugging | `06`, `01` |
| Sampling | `01` |
| PII и Security | `01`, `04`, `05` |
| PHP, Laravel, Symfony | `01` |
| Common Pitfalls и Tradeoffs | `01`, `06`, `07` |
| Senior-Level Short Answers | `01` |
| Mini-Practice | `01`, `06` |
| Чеклисты | `01`, `06` |
| Self-Check Questions | `01` |
| Ссылки | `01` |

Проверка покрытия: все ключевые понятия сохранены: observability/monitoring, logs/metrics/traces, structured logs, trace/correlation ID, OTel, Sentry, RED/USE, SLI/SLO/SLA, alerting, dashboards, incident debugging, sampling, PII, PHP/Laravel/Symfony integration, practice/self-check/links.

## Source: HTTP DNS TLS Nginx reverse proxy timeouts.md

| Старый раздел | Новый файл |
| --- | --- |
| Карта запроса | `02` |
| DNS basics | `02`, `03` |
| TCP и TLS | `02`, `03` |
| HTTP/1.1 vs HTTP/2 vs HTTP/3 | `02` |
| Keep-Alive | `02`, `07` |
| HTTP headers | `02` |
| Caching headers | `02`, `05` |
| Cookies | `02` |
| CORS | `02` |
| Reverse proxy | `02` |
| Load balancing | `02`, `04`, `06` |
| Nginx + PHP-FPM/FastCGI | `02`, `03` |
| Buffering | `02`, `05` |
| Gzip и Brotli | `02`, `05` |
| Upload limits | `02`, `05`, `07` |
| Timeouts | `07`, `02`, `06` |
| Retries | `07`, `02` |
| Status codes | `02`, `07` |
| Rate limiting | `02`, `07`, `06` |
| Troubleshooting 499, 502, 504 | `02`, `03`, `06`, `07` |
| Практический алгоритм диагностики | `06`, `02` |
| Типовые senior-вопросы и ответы | `02`, `07` |
| Pitfalls для интервью | `02`, `07` |
| Self-check | `02`, `07` |
| Mini-practice | `02`, `07`, `06` |
| Checklist перед интервью | `02`, `06` |
| Ссылки | `02`, `07` |

Проверка покрытия: сохранены все сетевые и HTTP-темы: DNS records/TTL/split-horizon, TCP/TLS/SNI/ALPN, HTTP versions, keep-alive, headers/cache/cookies/CORS, reverse proxy, load balancing, Nginx/FastCGI/PHP-FPM, buffering/compression/upload limits, timeouts/retries/status codes/rate limiting, 499/502/504, Nginx log variables, practice/self-check/links.

## Source: Linux production debugging for backend.md

| Старый раздел | Новый файл |
| --- | --- |
| Ментальная модель инцидента | `06`, `03` |
| Быстрый triage | `03`, `06` |
| Процессы: ps/top/htop | `03` |
| CPU и load average | `03` |
| Память/free/OOM killer | `03`, `06` |
| Диск/df/du/lsof/I/O | `03`, `06` |
| File descriptors, lsof, ulimit | `03` |
| Сеть: ss/netstat/curl/dig/tcpdump | `03`, `02`, `06` |
| Логи | `03`, `01`, `06` |
| systemd | `03`, `06` |
| cron | `03`, `06` |
| Permissions | `03`, `04` |
| strace | `03` |
| PHP-FPM worker exhaustion | `03`, `02`, `06` |
| Docker | `03`, `04` |
| Частые инциденты: Disk full | `03`, `06` |
| Частые инциденты: High load | `03`, `06` |
| Частые инциденты: Slow network/API timeout | `03`, `06`, `07` |
| Частые инциденты: PHP-FPM saturation | `03`, `02`, `06` |
| Частые инциденты: OOM | `03`, `06` |
| Типовые ловушки | `03`, `06` |
| Senior answers | `03` |
| Мини-практика | `03` |
| Self-check | `03` |
| Ссылки | `03` |

Проверка покрытия: сохранены команды и диагностические модели по процессам, CPU/load, memory/OOM, disk/inode/I/O, fd, network, logs, systemd, cron, permissions, strace, PHP-FPM, Docker, типовым инцидентам, pitfalls, senior answers, practice/self-check/links.

## Source: CI CD Docker GitLab deploy rollback.md

| Старый раздел | Новый файл |
| --- | --- |
| CI/CD: суть и принципы | `04` |
| GitLab CI: stages, jobs, runners, artifacts, cache | `04` |
| Типовой PHP pipeline | `04` |
| Composer install в CI и production | `04` |
| Dockerfile best practices для PHP | `04` |
| Docker Compose для dev environment | `04`, `03`, `05` |
| Secrets и env | `04`, `01`, `05` |
| Deploy: общая модель | `04`, `06` |
| Database migrations during deploy | `04`, `06` |
| Zero-downtime deploy | `04`, `06` |
| Blue-green, canary, rolling deploy | `04`, `06` |
| Feature environments | `04` |
| Rollback strategy | `04`, `06` |
| Release notes, tagging, versioning | `04`, `01`, `06` |
| Observability during deploy | `04`, `01`, `06` |
| Security scans и dependency audit | `04` |
| Практические объяснения для интервью | `04` |
| Common pitfalls and tradeoffs | `04`, `06` |
| Mini-practice | `04`, `06` |
| Self-check questions | `04` |
| Checklists | `04`, `06` |
| Ссылки | `04` |

Проверка покрытия: сохранены CI/CD определения, GitLab primitives, artifacts/cache, PHP pipeline, Composer, Dockerfile, Compose, secrets/env, deploy model, safe migrations, zero-downtime, rollout strategies, feature environments, rollback, release traceability, deploy observability, security scans, pitfalls, practice/checklists/links.

## Source: Object Storage S3 MinIO CDN media files.md

| Старый раздел | Новый файл |
| --- | --- |
| Что такое object storage | `05` |
| S3 как де-факто стандарт | `05` |
| Bucket, key, prefix, metadata | `05` |
| Консистентность S3 | `05` |
| Public и private files | `05` |
| Presigned URLs | `05`, `07` |
| Multipart upload | `05`, `07` |
| Lifecycle policies | `05` |
| Versioning | `05` |
| Encryption | `05` |
| Access policies и безопасность | `05`, `04` |
| CDN и object storage | `05`, `02` |
| Signed CDN URLs и cookies | `05` |
| Media processing pipeline | `05`, `07` |
| Thumbnails и derivatives | `05` |
| Virus scanning | `05` |
| Large uploads | `05`, `07`, `02` |
| MinIO | `05`, `04` |
| Backups и disaster recovery | `05`, `06` |
| Стоимость | `05` |
| PHP integration | `05` |
| Типичные pitfalls | `05` |
| Senior answers на интервью | `05` |
| Архитектурный шаблон для SaaS media storage | `05` |
| Self-check | `05` |
| Mini-practice | `05` |
| Checklist | `05` |
| Ссылки | `05` |

Проверка покрытия: сохранены object storage basics, S3 entities, bucket/key/metadata, consistency, access models, presigned/multipart uploads, lifecycle/versioning/encryption/IAM, CDN/cache/signed access, media pipeline, derivatives, virus scanning, large uploads, MinIO, backup/DR, cost model, PHP/Laravel/Symfony integration, pitfalls, senior answers, SaaS template, practice/checklist/links.

## Сквозные темы и где теперь искать

| Тема | Новый файл |
| --- | --- |
| Общий incident flow | `06` |
| Observability при инциденте | `01`, `06` |
| Deploy observability | `04`, `06`, `01` |
| Rollback decision | `04`, `06` |
| Timeout budget | `07`, `02` |
| Retries/backoff/jitter | `07`, `02`, `05` |
| Idempotency | `07`, `04`, `05` |
| Backpressure/rate limiting | `07`, `02`, `06` |
| PHP-FPM saturation | `03`, `02`, `06` |
| 499/502/504 | `02`, `06`, `03`, `07` |
| Docker build vs Docker debugging | `04`, `03` |
| Secrets/PII | `01`, `04`, `05` |
| CDN/cache/immutable assets | `05`, `02`, `04` |
| Large uploads | `05`, `07`, `02` |
| Queue workers | `01`, `04`, `05`, `07`, `06` |

## Финальная сверка

- Старые файлы не удалены и могут использоваться как источник полной истории.
- Все старые разделы перечислены в таблицах выше.
- Для каждого старого раздела указан как минимум один новый файл.
- Повторяющиеся темы вынесены в сквозные файлы `06` и `07`, а в профильных файлах оставлен контекст конкретной области.
- Блоки ссылок сохранены и расширены дополнительными источниками: AWS Builders Library, Stripe idempotency, Google SRE incident/postmortem/overload, Trivy, Semgrep, AWS SDK for PHP.
