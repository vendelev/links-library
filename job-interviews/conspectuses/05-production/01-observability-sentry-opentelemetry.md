# Observability, Sentry, OpenTelemetry

Конспект о том, как объяснять наблюдаемость на интервью, проектировать telemetry и связывать logs, metrics, traces, Sentry и релизы.

## Карта темы

- Observability отвечает на вопрос: почему система ведет себя так даже для заранее неизвестных проблем.
- Monitoring отвечает на вопрос: сломалось ли то, что мы заранее решили проверять.
- Базовые сигналы: logs, metrics, traces.
- Production-ready observability требует не только инструментов, но и дисциплины: service name, environment, release, trace/correlation ID, SLO, actionable alerts, runbooks, ownership.
- Частые инструменты в PHP: Sentry, OpenTelemetry, Prometheus, Grafana, Loki/ELK/OpenSearch, Datadog, New Relic.

Senior-ответ:

> Monitoring говорит, что checkout начал отдавать 500. Observability помогает понять, что 500 появились только после релиза, только для EUR-платежей через конкретного провайдера, где upstream latency выросла с 200 ms до 3 s.

## Logs, metrics, traces

### Logs

Логи - дискретные события: ошибка, бизнес-событие, состояние выполнения, решение кода.

Использовать для:

- деталей ошибки;
- audit trail;
- редких событий;
- отладки конкретного request/job;
- объяснения ветвлений бизнес-логики.

Не использовать для:

- высокочастотных счетчиков вместо метрик;
- хранения PII без маскирования;
- огромных payload целиком;
- единственного источника SLO.

### Metrics

Метрики - числовые временные ряды: request duration, error rate, queue size, memory usage.

Использовать для:

- алертов;
- SLI/SLO;
- трендов;
- capacity planning;
- быстрых health dashboards.

Типы:

- Counter: только растет, например `http_requests_total`.
- Gauge: текущее значение, например `queue_depth`.
- Histogram: распределение значений, например latency buckets.
- Summary: квантили на стороне клиента; осторожно при агрегации между инстансами.

### Traces

Трейсы показывают путь одного запроса через сервисы и операции. Trace состоит из span-ов.

Использовать для:

- distributed tracing;
- поиска медленного участка;
- анализа fan-out/fan-in;
- связи HTTP request, DB query, Redis call, queue job, external API call.

Не использовать как замену логам: traces обычно sampled, а error log должен сохраняться надежнее.

## Структурированное логирование

Структурированный лог - это событие с полями, обычно JSON, а не строка без стабильной схемы.

```json
{
  "level": "error",
  "message": "Payment provider request failed",
  "service": "billing-api",
  "env": "prod",
  "release": "2026.06.29-1",
  "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
  "span_id": "00f067aa0ba902b7",
  "correlation_id": "req-01J2...",
  "order_id": 12345,
  "provider": "stripe",
  "duration_ms": 742,
  "exception_class": "GatewayTimeoutException"
}
```

Практики:

- логировать событие, а не просто текст;
- использовать стабильные имена полей;
- добавлять `service`, `env`, `release`, `trace_id`, `span_id`, `correlation_id`;
- не логировать пароли, токены, карты, raw cookies, полные персональные данные, большие raw payload;
- нормализовать уровни `debug`, `info`, `warning`, `error`, `critical`;
- не писать `error`, если это ожидаемый бизнес-флоу.

Типичная ошибка:

> Логируют `json_encode($request->all())` и случайно отправляют в ELK/Sentry персональные данные, токены и payload на десятки килобайт.

## Correlation ID и Trace ID

- Correlation ID - прикладной идентификатор для связывания событий одного бизнес-запроса. Часто приходит из gateway или создается на входе.
- Trace ID - идентификатор распределенного trace по стандарту tracing-системы, например W3C Trace Context.
- Они могут совпадать по назначению, но не обязаны быть одним и тем же.

Практический подход:

- на входе HTTP request читать `traceparent`, `x-request-id`, `x-correlation-id`;
- если ID нет, создать;
- пробрасывать ID во все outgoing HTTP calls, messages, jobs, cron/CLI контексты;
- добавлять ID в логи, Sentry scope/tags, trace context;
- возвращать request ID в response header для поддержки.

Senior-ответ:

> Я разделяю trace ID как часть distributed tracing и correlation ID как бизнес/операционный идентификатор. Важно не название header-а, а сквозное распространение через HTTP, очереди и jobs плюс автоматическое добавление ID в логи и ошибки.

## Distributed tracing

- Trace - дерево или DAG операций одного запроса.
- Span - отдельная операция: HTTP handler, SQL query, Redis call, external API, queue publish, queue consume.

Ключевые поля span:

- `trace_id`;
- `span_id`;
- `parent_span_id`;
- `name`;
- `start_time`, `duration`;
- `status`;
- attributes: `http.method`, `http.route`, `db.system`, `messaging.system`, `error.type`.

Что объяснить на интервью:

- trace показывает critical path latency;
- parent-child связи строятся через context propagation;
- async jobs требуют явного сохранения context в message headers/payload;
- без sampling tracing может стать слишком дорогим;
- слишком детальные spans создают шум и overhead.

Пример:

> Если `/checkout` медленный, trace покажет, что controller занял 20 ms, DB - 40 ms, а внешний payment provider - 1800 ms. Метрика покажет деградацию p95, trace покажет причину, лог даст детали ошибки и безопасные payload identifiers.

## OpenTelemetry

OpenTelemetry - vendor-neutral стандарт и набор SDK/инструментов для сбора telemetry data: traces, metrics, logs.

Основные концепции:

- Tracer создает spans.
- Span описывает операцию внутри trace.
- Context propagation переносит trace context между функциями, HTTP, очередями, CLI/job.
- Exporter отправляет данные наружу: OTLP, Jaeger, Zipkin, Prometheus, vendor exporter.
- Collector принимает telemetry, фильтрует, батчит, семплирует, обогащает и отправляет в backend.
- Resource описывает источник telemetry: service name, version, environment, instance id.
- Instrumentation бывает ручной и автоматической.

Типовой pipeline:

```text
PHP app -> OpenTelemetry SDK/auto-instrumentation -> OTLP exporter -> OTel Collector -> Sentry/Grafana Tempo/Jaeger/Datadog/etc.
```

Почему Collector полезен:

- decoupling приложения от vendor-а;
- централизованный sampling/filtering;
- batch/retry;
- enrichment resource attributes;
- меньше конфигов в каждом сервисе;
- возможность менять routing/sampling без релиза приложения.

Context propagation:

- стандартный формат W3C Trace Context: `traceparent`, `tracestate`;
- baggage передается через `baggage`, но туда нельзя класть PII и большие значения.

Где propagation часто ломается:

- queue messages;
- async event bus;
- cron commands;
- manual Guzzle/cURL clients;
- retries без сохранения headers;
- message broker headers, которые framework не копирует;
- boundary между nginx/API gateway/app.

## Sentry

Sentry - инструмент для error tracking, performance monitoring, release health и issue management.

Полезные данные в error event:

- exception class/message;
- stacktrace;
- request URL/route;
- user id или anonymized id;
- tags: `env`, `release`, `service`, `tenant`, `feature`;
- breadcrumbs;
- trace context;
- custom context без PII.

Практики:

- настраивать `environment` и `release`;
- включать source context, но не отправлять секреты;
- добавлять tags для фильтрации;
- не ловить исключение и не делать только `captureMessage`, теряя stacktrace;
- не отправлять ожидаемые validation/business errors как exceptions;
- настраивать `before_send` для scrub PII.

Performance:

- Sentry Performance собирает transactions/spans и связывает ошибки с trace;
- важны `traces_sample_rate` или `traces_sampler`;
- profiling включать только при оправданной стоимости;
- performance issues должны быть связаны с release и deploy.

Release tracking помогает ответить:

- какая версия внесла ошибку;
- вырос ли crash/error rate после деплоя;
- какие commits связаны с issue;
- затронуты ли конкретные environments.

Senior-ответ:

> Для Sentry обязательны `environment`, `release`, корректный stacktrace, scrub PII, correlation/trace id и понятные tags. Без release Sentry превращается в корзину ошибок, а не в инструмент анализа регрессий.

## RED, USE, SLI, SLO, SLA

RED подходит для request-driven сервисов:

- Rate: сколько запросов в секунду.
- Errors: доля ошибок.
- Duration: latency, обычно p50/p95/p99.

Пример API metrics:

- `http_requests_total{route,method,status}`;
- `http_request_duration_seconds_bucket{route,method}`;
- error rate по `5xx`;
- p95 latency по route.

USE подходит для ресурсов:

- Utilization: насколько ресурс занят.
- Saturation: очередь/ожидание.
- Errors: ошибки ресурса.

Пример USE:

- CPU utilization;
- memory usage;
- disk IO wait;
- DB connection pool saturation;
- queue depth;
- Redis evictions/errors.

SLI/SLO/SLA:

- SLI - измеримый индикатор: availability, p95 latency, successful checkout rate.
- SLO - внутренняя цель по SLI, например 99.9% successful requests за 30 дней.
- SLA - внешний договор с последствиями: компенсации, штрафы, обязательства.
- Error budget - допустимая доля ошибок; если budget быстро сгорает, команда снижает риск: freeze deploys, reliability work, rollback.

Типичная ошибка:

> Алертить на каждую 500. Лучше алертить на burn rate/error budget, а отдельные ошибки отправлять в Sentry без пробуждения on-call.

## Alerting и dashboards

Хороший алерт:

- actionable;
- имеет owner;
- содержит impact;
- содержит ссылку на dashboard/runbook;
- не срабатывает на известный шум;
- связан с пользовательским влиянием или риском SLO.

Плохой алерт:

- `CPU > 80%` без контекста;
- одна ошибка в логах;
- flaky healthcheck;
- нет инструкции, что делать;
- дублируется в 5 системах.

Практические алерты:

- high error rate;
- p95/p99 latency regression;
- queue lag/depth too high;
- DB connection saturation;
- payment success rate drop;
- no events received для поточной системы;
- error budget burn rate.

Минимальный backend dashboard:

- traffic: RPS, active users/requests;
- errors: 5xx rate, domain failure rate;
- latency: p50/p95/p99 по важным routes;
- saturation: DB pool, queue depth, worker utilization;
- dependencies: external API latency/error rate;
- releases/deploy markers;
- links to logs/traces/Sentry.

Ошибки dashboard-ов:

- слишком много графиков без owner-а;
- average latency вместо percentiles;
- нет фильтра по route/service/env;
- нет связи с релизами;
- dashboard не используется при инцидентах.

## Sampling, cost, cardinality

Sampling снижает стоимость и overhead telemetry.

Виды:

- Head-based: решение принять trace принимается в начале запроса.
- Tail-based: решение принимается после просмотра trace; можно сохранить ошибки/медленные запросы.
- Probabilistic: случайный процент.
- Rule-based: по route, status, user segment, environment.

Правила:

- ошибки и slow traces сохранять с большим приоритетом;
- healthcheck/static endpoints семплировать агрессивно или исключать;
- low traffic critical flows можно сохранять 100%;
- high-cardinality attributes не использовать как labels в метриках;
- sampling должен быть консистентным по trace, иначе теряется дерево.

Pitfall:

> Семплировать каждый сервис независимо. В результате root span есть, downstream span потерян, и trace бесполезен.

## PII и security

Не отправлять без строгой необходимости:

- passwords;
- access/refresh tokens;
- session cookies;
- card data;
- passport/medical data;
- raw request/response bodies;
- authorization headers;
- private keys/secrets;
- email/phone/IP без политики обработки.

Практики:

- scrub/mask на уровне SDK и Collector;
- allowlist вместо blacklist для context fields;
- разные retention policies;
- RBAC в Sentry/Grafana/logs;
- audit доступа;
- separate environments;
- не использовать baggage для PII;
- проверять compliance: GDPR, PCI DSS, локальные требования.

Senior-ответ:

> Observability не должна становиться теневым хранилищем персональных данных. Я предпочитаю allowlist полей, scrub на SDK/Collector уровне и отдельную retention/access policy для logs, traces и error tracking.

## PHP, Laravel, Symfony

Общие PHP-заметки:

- PHP-FPM request lifecycle короткий, поэтому batch/export должен быть завершен в конце request.
- Для CLI workers/queues нужно flush telemetry при завершении job/process.
- Long-running workers требуют очистки context между jobs, иначе возможна утечка trace/user context.
- OPcache/preload обычно не проблема, но auto-instrumentation нужно проверять в конкретном runtime.
- Guzzle/cURL, PDO, Redis, AMQP/Kafka клиенты часто требуют instrumentation или middleware.
- Fatal errors/shutdown handler должны попадать в error tracking.

Laravel:

- middleware для correlation ID и trace headers;
- Monolog processors для `trace_id`, `span_id`, `correlation_id`;
- Sentry Laravel SDK для exceptions, breadcrumbs, performance;
- queue middleware для propagation context в job payload/headers;
- events/listeners для бизнес-метрик;
- Telescope не заменяет production observability;
- Horizon полезен для queue metrics, но не заменяет SLO и traces;
- не логировать `$request->all()`;
- route name лучше raw URL с IDs для снижения cardinality;
- очищать Sentry scope/context в long-running workers;
- связывать job failure с trace/correlation ID исходного запроса.

Symfony:

- event subscriber/kernel middleware для request ID;
- Monolog processors;
- Sentry Symfony bundle;
- Messenger middleware для context propagation;
- HttpClient/Guzzle instrumentation;
- Console commands и workers с явным flush;
- profiler полезен локально, но не заменяет production telemetry;
- route name как span/metric attribute вместо полного URL;
- Messenger retry должен сохранять trace/correlation context;
- normalizer/serializer ошибки не должны отправлять PII в logs/Sentry.

## Senior short answers

**Что такое observability?**

> Способность понять внутреннее состояние системы по telemetry signals: logs, metrics, traces. В отличие от monitoring, observability помогает исследовать неизвестные заранее проблемы.

**Что выбрать: logs, metrics или traces?**

> Метрики - для алертов и трендов, логи - для деталей событий, traces - для пути запроса и latency breakdown. В зрелой системе они связаны через trace/correlation ID.

**Зачем OpenTelemetry, если есть Sentry/Datadog?**

> OpenTelemetry дает vendor-neutral instrumentation и единый формат. Приложение отправляет OTLP в Collector, а Collector маршрутизирует в Sentry, Tempo, Datadog или другой backend.

**Как проектировать алерты?**

> От пользовательского impact и SLO: error rate, latency, success rate, queue lag. Алерт должен быть actionable, с owner, dashboard и runbook.

**Как снизить стоимость observability?**

> Sampling, retention policies, агрегация метрик, контроль cardinality, исключение healthcheck/noisy endpoints, Collector-side filtering и scrub.

**Как связать ошибку в Sentry с логами?**

> Добавить `trace_id`/`correlation_id` в Sentry scope/tags и structured logs. Тогда issue в Sentry ведет к trace и связанным логам.

## Mini-practice

### API endpoint

Для `POST /api/orders/{id}/pay` опишите:

- RED metrics по route name `orders.pay`;
- spans: controller, validation, DB transaction, payment provider, event publish;
- logs: payment started/failed/succeeded с `order_id`, `provider`, `correlation_id`, без card/token;
- SLO: например 99.5% successful payment API under 1s excluding provider-declared downtime.

### Очередь

Для `SendInvoiceEmail`:

- сохранить `traceparent`/correlation ID в message headers/payload;
- metrics: queue depth, job duration, retries, failures, age of oldest message;
- alerts: growing lag, high failure rate, exhausted retries;
- Sentry: final failure с invoice/customer safe ID, release, worker name, trace context.

### Инцидент после релиза

Если выросли 502 на `/checkout`:

- проверить release marker;
- сравнить error rate до/после;
- открыть Sentry issues по release;
- найти trace для 502;
- проверить upstream spans;
- проверить логи по `trace_id`;
- rollback или feature flag off;
- добавить тест/алерт/instrumentation после postmortem.

## Checklist

- Есть service name, environment, release во всех сигналах.
- Logs структурированы и содержат trace/correlation ID.
- Metrics покрывают RED для API и USE для ресурсов.
- Traces проходят через HTTP, queues и workers.
- Sentry настроен с release, environment, tags и PII scrub.
- Dashboards отражают пользовательский impact.
- Alerts actionable и привязаны к SLO/runbook.
- High-cardinality labels запрещены в metrics.
- Sampling и retention осознанно настроены.
- Есть incident process и postmortem loop.

## Self-check

1. Чем observability отличается от monitoring на практическом примере?
2. Почему нельзя использовать `user_id` как Prometheus label?
3. Что такое `traceparent` и зачем он нужен?
4. Чем correlation ID отличается от trace ID?
5. Почему average latency хуже p95/p99 для алертов?
6. Что должен содержать хороший Sentry event?
7. Когда нужен OpenTelemetry Collector?
8. Как пробросить trace context через очередь?
9. Что такое error budget и как он влияет на релизы?
10. Какие PII-риски есть у logs/traces/Sentry?
11. Почему healthcheck endpoint часто исключают из tracing или сильно семплируют?
12. Какие метрики нужны для Laravel queue workers?

## Ссылки

- [OpenTelemetry Docs](https://opentelemetry.io/docs/)
- [OpenTelemetry PHP](https://opentelemetry.io/docs/languages/php/)
- [W3C Trace Context](https://www.w3.org/TR/trace-context/)
- [W3C Baggage](https://www.w3.org/TR/baggage/)
- [Sentry Docs](https://docs.sentry.io/)
- [Sentry PHP](https://docs.sentry.io/platforms/php/)
- [Sentry Laravel](https://docs.sentry.io/platforms/php/guides/laravel/)
- [Sentry Symfony](https://docs.sentry.io/platforms/php/guides/symfony/)
- [Prometheus Docs](https://prometheus.io/docs/introduction/overview/)
- [Grafana Docs](https://grafana.com/docs/)
- [Grafana Loki Docs](https://grafana.com/docs/loki/latest/)
- [Elastic Observability](https://www.elastic.co/observability)
- [Google SRE Book](https://sre.google/sre-book/table-of-contents/)
- [Google SRE Workbook](https://sre.google/workbook/table-of-contents/)
- [Google SRE: Monitoring Distributed Systems](https://sre.google/sre-book/monitoring-distributed-systems/)
- [Google SRE: Service Level Objectives](https://sre.google/sre-book/service-level-objectives/)
- [Martin Fowler: Observability](https://martinfowler.com/tags/observability.html)
- [Honeycomb Observability Resources](https://www.honeycomb.io/resources)
