# Production incident playbook

Сквозной конспект по диагностике production-инцидентов. Объединяет observability, HTTP/Nginx, Linux debugging, CI/CD rollback и object storage в единый алгоритм.

## Главная идея

На интервью Senior должен показать не «я перезапущу сервис», а управляемый процесс:

1. Зафиксировать impact и scope.
2. Быстро сузить слой проблемы.
3. Найти безопасный mitigation.
4. Не потерять evidence.
5. После стабилизации закрыть gap в observability, тестах, runbook или архитектуре.

Senior-ответ:

> Я начинаю с impact: кто страдает, какой endpoint/job/business flow, когда началось и связано ли с релизом. Затем смотрю метрики, логи, traces и системные сигналы, чтобы выбрать безопасный mitigation: rollback, feature flag off, scale, rate limit, отключение тяжелой job или degraded mode.

## Первые вопросы

1. Что сломалось: endpoint, worker, cron, весь хост, контейнер, сеть, диск, БД, object storage, CDN, external API.
2. Когда началось: релиз, deploy, миграция, cron, рост трафика, ротация логов, сертификат, DNS, lifecycle rule.
3. Насколько широко: один host/pod/container, один AZ/DC/region, одна версия, один tenant, все пользователи.
4. Что ограничено: CPU, память, диск, inode, сеть, file descriptors, PHP-FPM workers, DB connections, queue lag, external API latency, S3/CDN errors.
5. Какой mitigation безопасен: rollback, scale out, restart конкретного сервиса, stop worker, feature flag, rate limit, cleanup disk, increase limits, direct-to-S3 upload, CDN invalidation.

## Общий алгоритм

1. Проверить impact: пользовательский сценарий, business metric, endpoint/job.
2. Проверить timeline: deploy marker, release, migration, cron, traffic spike, infra change.
3. Открыть RED dashboard: rate, errors, duration.
4. Проверить saturation: CPU, memory, disk I/O, DB pool, queue depth, PHP-FPM workers.
5. Открыть Sentry issues по `release`, `env`, `service`.
6. Найти trace медленного/ошибочного запроса.
7. По `trace_id`/`correlation_id` открыть связанные логи.
8. Проверить dependencies: DB, Redis, queue, external APIs, object storage, CDN.
9. Принять mitigation: rollback, feature flag off, degrade gracefully, scale, hotfix.
10. После mitigation оформить postmortem и добавить недостающие сигналы/тесты/runbook.

## Минимальный triage на host/container

```bash
date
hostname
uptime
top
free -h
df -h
df -ih
ss -s
systemctl --failed
journalctl -p warning..alert --since "30 min ago"
```

Если Docker:

```bash
docker ps
docker logs --tail=200 <container>
docker stats
docker inspect <container> --format '{{.State.OOMKilled}} {{.State.ExitCode}} {{.State.RestartCount}}'
```

## Диагностика по симптомам

### Выросли 5xx после deploy

Проверить:

- release marker и changelog;
- Sentry issues по release;
- error rate до/после;
- slow/error traces;
- logs новой версии;
- миграции и совместимость схемы;
- queue workers старой версии;
- health/readiness;
- business metrics.

Mitigation:

- остановить rollout;
- выключить feature flag;
- canary rollback;
- redeploy previous image digest;
- forward-fix, если rollback опасен из-за данных/миграции.

### Вырос p95/p99 latency

Проверить:

- route/env/region/provider/tenant scope;
- RED dashboard;
- slow trace и самый дорогой span;
- DB slow queries, locks, connection pool;
- Redis/external API latency;
- PHP-FPM worker saturation;
- queue lag;
- network DNS/connect/TLS/TTFB через `curl -w`.

Mitigation:

- timeout budget;
- circuit breaker/degradation;
- scale out;
- отключить тяжелый endpoint/job;
- rollback/hotfix.

### 502 от Nginx к PHP-FPM

Проверить:

- Nginx error log: `connect() failed`, `upstream prematurely closed connection`, `recv() failed`, `upstream sent too big header`;
- статус PHP-FPM;
- socket/path/permissions;
- `pm.max_children`;
- OOM killer;
- slowlog;
- deploy/restart события.

Mitigation:

- вернуть предыдущий release;
- scale out;
- временно поднять workers только после memory sizing;
- исправить socket/permissions/config;
- снизить concurrency или отключить тяжелый flow.

### 504 Gateway Timeout

Проверить:

- `$upstream_response_time` близок к `fastcgi_read_timeout`/`proxy_read_timeout`;
- PHP slowlog;
- DB slow query log;
- locks/deadlocks;
- external API latency;
- PHP-FPM queue;
- timeout chain: client, CDN, LB, Nginx, PHP-FPM, app clients.

Mitigation:

- не просто увеличивать timeout;
- вынести long-running работу в queue;
- вернуть `202 Accepted` и status endpoint;
- добавить idempotency key;
- ограничить retries и общий timeout budget.

### 499 Client Closed Request

Проверить:

- `$request_time`;
- `$upstream_response_time`;
- user agent;
- endpoint;
- размер ответа;
- timeouts на CDN/LB/client;
- backend latency.

Вывод:

> 499 часто симптом медленного backend или короткого client timeout, но не всегда ошибка сервера.

### Disk full

Проверить:

- `df -h`, `df -ih`;
- `/var/log`, `/tmp`, uploads, cache, Docker overlay;
- `du -xh --max-depth=1 /var | sort -h`;
- `lsof +L1` для deleted files;
- logrotate/retention.

Mitigation:

- безопасно освободить место;
- переоткрыть лог или перезапустить конкретный процесс, если удаленный файл держится fd;
- расширить volume;
- настроить retention.

### OOM / OOMKilled

Проверить:

- `journalctl -k | grep -i oom`;
- `docker inspect`/orchestrator events;
- memory limit контейнера;
- `free -h`, `vmstat 1`;
- топ процессов по памяти;
- PHP `memory_limit`, FPM `pm.max_children`, batch size, недавний релиз.

Mitigation:

- уменьшить concurrency;
- rollback;
- поднять лимит;
- разбить batch;
- включить streaming;
- уменьшить FPM workers, если `pm.max_children * RSS` не помещается.

### Slow external API / network

Проверить:

- `curl -w` для DNS/connect/TLS/TTFB/total;
- `dig` для DNS и resolvers;
- `ss -tan` для TCP states;
- `tcpdump` для handshake/retransmits/RST;
- proxy/firewall/security groups/route/IPv6;
- сравнить из контейнера и с хоста;
- retries/timeouts/circuit breaker.

### Object storage/CDN incident

Проверить:

- `403 AccessDenied` vs `404 NotFound` vs network timeout;
- bucket policy/IAM/KMS permissions;
- public/private bucket и Block Public Access;
- signed URL TTL и method/key;
- CloudFront OAC/OAI;
- CDN cache/invalidation/versioned key;
- lifecycle rule, versioning, delete marker;
- incomplete multipart uploads;
- egress/request/KMS costs;
- storage usage and hot objects.

Mitigation:

- вернуть policy/config;
- выдать новый signed URL;
- CDN invalidation для emergency;
- переключить на immutable key;
- restore object version;
- stop broken lifecycle rule;
- включить fallback/degraded mode.

## Runbook rollback

Runbook должен отвечать:

- где найти предыдущий release/image digest;
- кто принимает решение rollback;
- какие метрики являются trigger;
- какая команда rollback;
- как проверить health/smoke;
- как коммуницировать в incident channel;
- какие follow-up tasks создать.

Когда rollback хорош:

- предыдущий артефакт известен и доступен;
- миграции совместимы назад;
- side effects обратимы или не произошли;
- проблема явно в новом release.

Когда rollback плох:

- уже изменены данные в новом формате;
- external API получил необратимые side effects;
- старая версия несовместима с новой схемой;
- forward-fix безопаснее.

## Evidence before restart

Перед рестартом, если ситуация позволяет:

- `systemctl status <service>`;
- `journalctl -u <service> --since "30 min ago"`;
- PID процесса;
- CPU/memory/fd состояние;
- последние Nginx/PHP-FPM/app logs;
- OOM/core/segfault признаки;
- текущий release/image digest;
- sample request id / trace id.

## Postmortem loop

После стабилизации:

- описать timeline;
- описать impact;
- выделить root cause и contributing factors;
- отметить, что сработало и что не сработало;
- добавить недостающие метрики/алерты/traces/log fields;
- добавить тесты или safety checks в CI/CD;
- обновить runbook;
- назначить owners и сроки.

## Interview сценарии

### Checkout p95 вырос с 300 ms до 2 s

Ответ:

> Сначала определю scope и impact по метрикам: route, env, release, region, provider. Потом посмотрю deploy marker и Sentry. Далее возьму slow trace, найду дорогой span: DB, Redis, external payment, queue. По trace/correlation ID проверю логи. Если это регрессия релиза - rollback/feature flag. Если dependency - timeout/circuit breaker/degradation. После инцидента добавлю SLO alert, dashboard или instrumentation, если их не хватило.

### API возвращает 504, но операция в БД выполнена

Ответ:

> Proxy timeout истек раньше завершения PHP-кода или DB operation. Nginx закрыл клиентский запрос, но PHP/FPM мог продолжить выполнение. Нужны app-level timeout/cancellation, idempotency key, транзакции и корректный timeout budget.

### Пользователи за NAT получают 429

Ответ:

> Rate limit по IP слишком грубый. Лучше использовать user/API key/tenant key после authentication, IP оставить fallback. Проверить trusted proxy и реальный client IP.

### Пользователь обновил аватар, но CDN отдает старый

Ответ:

> Если key mutable, CDN/browser cache может отдавать старую версию. Лучшее решение - immutable keys с ULID/hash/version, обновление active key в БД и долгий `Cache-Control`. Invalidation оставить как fallback.

## Checklist production readiness

- Есть service name, environment, release во всех сигналах.
- Logs структурированы и содержат trace/correlation ID.
- Metrics покрывают RED и USE.
- Traces проходят через HTTP, queues и workers.
- Sentry настроен с release, environment, tags и PII scrub.
- Dashboards отражают пользовательский impact.
- Alerts actionable и привязаны к SLO/runbook.
- Deploy annotations есть в мониторинге.
- Rollback runbook проверен.
- Миграции backward-compatible.
- PHP-FPM sizing учитывает memory per worker.
- Timeouts/retries/idempotency спроектированы budget-based.
- Object storage/CDN имеют private-by-default, lifecycle, versioning/backup strategy.

## Ссылки

- [Google SRE Book](https://sre.google/sre-book/table-of-contents/)
- [Google SRE Workbook](https://sre.google/workbook/table-of-contents/)
- [Google SRE: Managing Incidents](https://sre.google/sre-book/managing-incidents/)
- [Google SRE: Postmortem Culture](https://sre.google/sre-book/postmortem-culture/)
- [Atlassian Incident Management](https://www.atlassian.com/incident-management)
- [OpenTelemetry Docs](https://opentelemetry.io/docs/)
- [Sentry Docs](https://docs.sentry.io/)
- [Nginx docs](https://nginx.org/en/docs/)
- [Docker CLI reference](https://docs.docker.com/reference/cli/docker/)
- [GitLab Environments and Deployments](https://docs.gitlab.com/ci/environments/)
