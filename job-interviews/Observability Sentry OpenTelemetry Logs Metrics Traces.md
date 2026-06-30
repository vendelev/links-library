# Observability, Sentry, OpenTelemetry, Logs, Metrics, Traces

Конспект для Lead/Senior PHP backend developer: как объяснять наблюдаемость на интервью, проектировать сбор сигналов, отлаживать инциденты и избегать типичных ошибок.

## Короткая карта темы

- **Observability** отвечает на вопрос: "почему система ведет себя так?" даже для заранее неизвестных проблем.
- **Monitoring** отвечает на вопрос: "сломалось ли то, что мы заранее решили проверять?".
- Три базовых сигнала: **logs**, **metrics**, **traces**.
- В production важны не только инструменты, а дисциплина: единые ID, структурированные события, SLO, алерты, runbook, ownership.
- Для PHP чаще всего встречаются: Sentry, OpenTelemetry SDK/auto-instrumentation, Prometheus, Grafana, Loki/ELK/OpenSearch, Datadog/New Relic.

## Observability vs Monitoring

**Monitoring** - заранее заданные проверки, дашборды и алерты: CPU, memory, HTTP 5xx, latency, queue depth, DB connections.

**Observability** - способность исследовать внутреннее состояние системы по внешним сигналам без деплоя нового кода. Особенно важна в микросервисах, очередях, асинхронных интеграциях и high-load системах.

Практическое объяснение на интервью:

> Monitoring говорит, что checkout начал отдавать 500. Observability помогает понять, что 500 появились только для платежей через конкретного провайдера, после релиза `2026.06.29`, при запросах с `currency=EUR`, где upstream latency выросла с 200 ms до 3 s.

Tradeoff:

- Monitoring проще и дешевле.
- Observability дороже по storage/cardinality/инструментированию, но быстрее сокращает MTTR.

## Logs, Metrics, Traces

### Logs

Логи - дискретные события: ошибка, бизнес-событие, состояние выполнения, решение кода.

Использовать для:

- деталей ошибки;
- audit trail;
- редких событий;
- отладки конкретного запроса или job;
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
- быстрых health-check дашбордов.

Типы метрик:

- **Counter**: только растет, например `http_requests_total`.
- **Gauge**: текущее значение, например `queue_depth`.
- **Histogram**: распределение значений, например latency buckets.
- **Summary**: квантиль на стороне клиента; осторожно при агрегации между инстансами.

### Traces

Трейсы показывают путь одного запроса через сервисы и операции. Trace состоит из span-ов.

Использовать для:

- distributed tracing;
- поиска медленного участка;
- понимания зависимостей;
- анализа fan-out/fan-in;
- связи HTTP request, DB query, queue job, external API call.

Не использовать как замену логам: trace обычно sampled, а лог ошибки должен сохраняться надежнее.

## Структурированное логирование

Структурированный лог - это событие с полями, обычно JSON, а не строка "User failed login".

Пример полей:

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

Хорошая практика:

- логировать событие, а не просто текст;
- использовать стабильные имена полей;
- добавлять `service`, `env`, `release`, `trace_id`, `correlation_id`;
- не логировать пароли, токены, карты, raw cookies, полные персональные данные;
- нормализовать error level: `debug`, `info`, `warning`, `error`, `critical`;
- не писать `error`, если это ожидаемый бизнес-флоу.

Типичная ошибка:

> Логируют `json_encode($request->all())`, случайно отправляя в ELK/Sentry персональные данные, токены и payload на десятки килобайт.

## Correlation ID и Trace ID

**Correlation ID** - прикладной идентификатор для связывания событий одного бизнес-запроса. Часто приходит из gateway или создается на входе.

**Trace ID** - идентификатор распределенного trace по стандарту tracing-системы, например W3C Trace Context.

Они могут совпадать по назначению, но не обязаны быть одним и тем же.

Практический подход:

- на входе HTTP request читать `traceparent` и `x-request-id`/`x-correlation-id`;
- если нет ID - создать;
- пробрасывать ID во все outgoing HTTP calls, messages, jobs;
- добавлять ID в логи, Sentry scope, trace context;
- возвращать request ID в response header для поддержки.

Senior-level ответ:

> Я разделяю trace ID как часть distributed tracing и correlation ID как бизнес/операционный идентификатор. Важно не название header-а, а сквозное распространение через HTTP, очереди и cron/jobs, плюс автоматическое добавление этих ID в логи и ошибки.

## Distributed Tracing

Trace - дерево или DAG операций одного запроса.

Span - отдельная операция: HTTP handler, SQL query, Redis call, external API, queue publish, queue consume.

Ключевые поля span:

- `trace_id`;
- `span_id`;
- `parent_span_id`;
- `name`;
- `start_time`, `duration`;
- `status`;
- attributes: `http.method`, `http.route`, `db.system`, `messaging.system`, `error.type`.

Что важно уметь объяснить:

- trace показывает critical path latency;
- parent-child связи строятся через context propagation;
- async jobs требуют явного сохранения context в message headers;
- без sampling tracing может стать слишком дорогим;
- слишком детальные spans создают шум и overhead.

Пример интервью-объяснения:

> Если `/checkout` медленный, trace покажет, что controller занял 20 ms, DB - 40 ms, а внешний payment provider - 1800 ms. Метрика покажет деградацию p95, trace покажет конкретную причину, лог даст детали ошибки или payload identifiers.

## OpenTelemetry

OpenTelemetry - vendor-neutral стандарт и набор SDK/инструментов для сбора telemetry data: traces, metrics, logs.

### Основные концепции

**Tracer** - объект, создающий spans.

**Span** - операция внутри trace.

**Context propagation** - перенос trace context между функциями, HTTP-запросами, очередями, CLI/job.

**Exporter** - компонент, отправляющий данные наружу: OTLP, Jaeger, Zipkin, Prometheus, vendor exporter.

**Collector** - отдельный процесс/agent/gateway, который принимает telemetry, фильтрует, батчит, семплирует, обогащает и отправляет в backend.

**Resource** - описание источника telemetry: service name, version, environment, instance id.

**Instrumentation** - ручная или автоматическая вставка spans/metrics/logs.

### Типовой pipeline

```text
PHP app -> OpenTelemetry SDK/auto-instrumentation -> OTLP exporter -> OTel Collector -> Sentry/Grafana Tempo/Jaeger/Datadog/etc.
```

Почему Collector полезен:

- decoupling приложения от vendor-а;
- централизованный sampling/filtering;
- batch/retry;
- enrichment resource attributes;
- меньше конфигов в каждом сервисе.

Senior-level ответ:

> Я предпочитаю отправлять telemetry из приложения в OpenTelemetry Collector по OTLP, а уже Collector настраивать под конкретные backend-ы. Это снижает vendor lock-in и позволяет менять sampling, batching и routing без релиза приложения.

### Context propagation

Стандартный формат - W3C Trace Context:

- `traceparent`;
- `tracestate`.

Для baggage:

- `baggage` header, но осторожно: это может увеличить размер запросов и случайно перенести чувствительные данные.

Практические места, где propagation часто ломается:

- queue messages;
- async event bus;
- cron commands;
- manual Guzzle/cURL clients;
- retries без сохранения headers;
- message broker headers, которые не копируются framework-ом;
- boundary между nginx/API gateway/app.

## Sentry

Sentry - инструмент для error tracking, performance monitoring, release health и issue management.

### Errors

Sentry группирует ошибки в issues по fingerprint/stacktrace. Полезные данные:

- exception class/message;
- stacktrace;
- request URL/route;
- user id или anonymized id;
- tags: `env`, `release`, `service`, `tenant`, `feature`;
- breadcrumbs;
- trace context;
- custom context без PII.

Практические рекомендации:

- настраивать `environment` и `release`;
- включать source context, но не отправлять секреты;
- добавлять tags для фильтрации;
- не ловить исключение и не делать только `captureMessage`, теряя stacktrace;
- не отправлять ожидаемые validation/business errors как exceptions;
- настраивать `before_send` для scrub PII.

### Performance

Sentry Performance может собирать transactions/spans и связывать ошибки с trace.

Важные настройки:

- `traces_sample_rate` или `traces_sampler`;
- profiling, если доступен и оправдан;
- performance issue detection;
- связка с release и deploy.

Tradeoff:

- 100% tracing удобно в dev/staging и для low traffic;
- в production high-load обычно нужен sampling;
- tail-based sampling лучше сохраняет интересные trace, но чаще требует Collector/backend support.

### Releases

Release tracking помогает ответить:

- какая версия внесла ошибку;
- вырос ли crash/error rate после деплоя;
- какие commits связаны с issue;
- затронуты ли конкретные environments.

Senior-level ответ:

> Для Sentry я считаю обязательными `environment`, `release`, корректный stacktrace, scrub PII, correlation/trace id и понятные tags. Без release Sentry превращается просто в корзину ошибок, а не в инструмент анализа регрессий.

## Metrics: RED и USE

### RED

Подходит для request-driven сервисов.

- **Rate**: сколько запросов в секунду.
- **Errors**: доля ошибок.
- **Duration**: latency, обычно p50/p95/p99.

Пример для API:

- `http_requests_total{route,method,status}`;
- `http_request_duration_seconds_bucket{route,method}`;
- error rate по `5xx`;
- p95 latency по route.

### USE

Подходит для ресурсов.

- **Utilization**: насколько ресурс занят.
- **Saturation**: очередь/ожидание.
- **Errors**: ошибки ресурса.

Пример:

- CPU utilization;
- memory usage;
- disk IO wait;
- DB connection pool saturation;
- queue depth;
- Redis evictions/errors.

Senior-level ответ:

> Для пользовательского API я начинаю с RED, для инфраструктурных ресурсов - с USE. Если есть только CPU и memory, но нет error rate и latency, это не production-ready observability для backend-сервиса.

## SLI, SLO, SLA

**SLI** - измеримый индикатор: availability, p95 latency, successful checkout rate.

**SLO** - внутренняя цель по SLI: 99.9% successful requests за 30 дней.

**SLA** - внешний договор с последствиями: компенсации, штрафы, обязательства.

Пример:

- SLI: доля HTTP requests с `status < 500` и latency `< 500 ms`.
- SLO: 99.5% за rolling 30 days.
- SLA: 99.0% monthly uptime для клиента.

Error budget:

- если SLO 99.9%, допустимо 0.1% ошибок;
- когда budget быстро сгорает, команда снижает риск: freeze deploys, фокус на reliability, rollback.

Типичная ошибка:

> Алертить на каждую 500 ошибку. Лучше алертить на burn rate/error budget, а отдельные ошибки отправлять в Sentry без пробуждения on-call.

## Alerting

Хороший алерт:

- actionable;
- имеет owner;
- содержит impact;
- содержит ссылку на dashboard/runbook;
- не срабатывает на известный шум;
- связан с пользовательским влиянием или риском SLO.

Плохой алерт:

- "CPU > 80%" без контекста;
- "одна ошибка в логах";
- flaky healthcheck;
- нет инструкции, что делать;
- дублируется в 5 системах.

Практические виды алертов:

- high error rate;
- p95/p99 latency regression;
- queue lag/depth too high;
- DB connection saturation;
- payment success rate drop;
- no events received, если система должна постоянно обрабатывать поток;
- error budget burn rate.

Senior-level ответ:

> Я стараюсь алертить не на симптомы инфраструктуры сами по себе, а на пользовательский impact и SLO. Инфраструктурные метрики полезны для диагностики, но не каждый высокий CPU должен будить человека ночью.

## Dashboards

Минимальный dashboard для backend-сервиса:

- traffic: RPS, active users/requests;
- errors: 5xx rate, domain failure rate;
- latency: p50/p95/p99 по важным routes;
- saturation: DB pool, queue depth, worker utilization;
- dependencies: external API latency/error rate;
- releases/deploy markers;
- links to logs/traces/Sentry.

Ошибки в dashboard-ах:

- слишком много графиков без owner-а;
- средняя latency вместо percentiles;
- нет фильтра по route/service/env;
- нет связи с релизами;
- dashboard не используется при инцидентах.

## Incident Debugging

Практический порядок анализа:

1. Проверить impact: кто страдает, какой endpoint/job/business flow.
2. Открыть RED dashboard: rate/errors/duration.
3. Проверить recent deploys/releases.
4. Открыть Sentry issues по release/env/service.
5. Найти trace медленного/ошибочного запроса.
6. По `trace_id`/`correlation_id` открыть связанные логи.
7. Проверить dependencies: DB, Redis, queue, external APIs.
8. Принять решение: rollback, feature flag off, degrade gracefully, scale, hotfix.
9. После mitigation оформить postmortem и добавить недостающие сигналы.

Мини-шаблон вопроса на интервью:

> У checkout вырос p95 с 300 ms до 2 s. Что делаете?

Короткий ответ:

> Сначала определю scope и impact по метрикам: route, env, release, region, provider. Потом посмотрю deploy marker и Sentry. Далее возьму slow trace, найду самый дорогой span: DB, Redis, external payment, queue. По trace/correlation ID проверю логи. Если это регрессия релиза - rollback/feature flag. Если dependency - timeout/circuit breaker/degradation. После инцидента добавлю SLO alert, dashboard или instrumentation, если их не хватило.

## Sampling

Sampling снижает стоимость и overhead telemetry.

Виды:

- **Head-based sampling**: решение принять trace принимается в начале запроса.
- **Tail-based sampling**: решение принимается после просмотра trace, можно сохранить ошибки/медленные запросы.
- **Probabilistic sampling**: случайный процент.
- **Rule-based sampling**: по route, status, user segment, environment.

Практические правила:

- ошибки и slow traces сохранять с большим приоритетом;
- healthcheck/static endpoints семплировать агрессивно или исключать;
- low traffic critical flows можно сохранять 100%;
- high-cardinality attributes не использовать как labels в метриках;
- sampling должен быть консистентным по trace, иначе теряется дерево.

Pitfall:

> Семплировать каждый сервис независимо. В результате root span есть, а downstream span потерян, и trace становится бесполезным.

## PII и Security

Что нельзя отправлять без строгой необходимости:

- passwords;
- access/refresh tokens;
- session cookies;
- card data;
- passport/medical data;
- raw request/response bodies;
- authorization headers;
- private keys/secrets;
- email/phone/IP без политики обработки.

Практики защиты:

- scrub/mask на уровне SDK и Collector;
- allowlist вместо blacklist для context fields;
- разные retention policies;
- RBAC в Sentry/Grafana/Logs;
- audit доступа;
- separate environments;
- не использовать baggage для PII;
- проверять compliance: GDPR, PCI DSS, локальные требования.

Senior-level ответ:

> Observability не должна становиться теневым хранилищем персональных данных. Я предпочитаю allowlist полей, scrub на SDK/Collector уровне и отдельную политику retention/access для logs, traces и error tracking.

## PHP, Laravel, Symfony

### Общие PHP-заметки

- PHP-FPM request lifecycle короткий, поэтому batch/export должен быть аккуратно завершен в конце request.
- Для CLI workers/queues важно flush telemetry при завершении job/process.
- Long-running workers требуют очистки context между jobs, иначе возможна утечка trace/user context.
- OPcache/preload обычно не проблема, но auto-instrumentation нужно проверять в конкретном runtime.
- Guzzle/cURL, PDO, Redis, AMQP/Kafka клиенты часто требуют instrumentation или middleware.
- Ошибки fatal/shutdown handler должны корректно попадать в error tracking.

### Laravel

Практические точки интеграции:

- middleware для correlation ID и trace headers;
- Monolog processors для добавления `trace_id`, `span_id`, `correlation_id`;
- Sentry Laravel SDK для exceptions, breadcrumbs, performance;
- queue middleware для propagation context в job payload/headers;
- events/listeners для бизнес-метрик;
- Telescope не заменяет production observability;
- Horizon полезен для queue metrics, но не заменяет SLO и traces.

На что обратить внимание:

- не логировать `$request->all()`;
- добавлять route name вместо raw URL с IDs, чтобы снизить cardinality;
- очищать Sentry scope/context в long-running workers;
- связывать job failure с trace/correlation ID исходного запроса.

### Symfony

Практические точки интеграции:

- event subscriber/kernel middleware для request ID;
- Monolog processors;
- Sentry Symfony bundle;
- Messenger middleware для context propagation;
- HttpClient/Guzzle instrumentation;
- Console commands и workers с явным flush;
- profiler полезен локально, но не заменяет production telemetry.

На что обратить внимание:

- route name как span/metric attribute вместо полного URL;
- Messenger retry должен сохранять trace/correlation context;
- normalizer/serializer ошибки не должны отправлять PII в logs/Sentry.

## Common Pitfalls и Tradeoffs

- **Логи без структуры**: сложно фильтровать и строить алерты.
- **Нет trace/correlation ID**: невозможно связать Sentry, logs и traces.
- **High cardinality labels**: `user_id`, `order_id`, raw URL в metrics labels взрывают стоимость и Prometheus storage.
- **Средние значения вместо percentiles**: average latency скрывает p95/p99 проблемы.
- **Алерты без действия**: alert fatigue и игнорирование on-call.
- **100% tracing без оценки стоимости**: высокий overhead и счета.
- **Слишком много spans**: шум, сложность анализа.
- **Нет release tracking**: сложно связать ошибки с деплоем.
- **PII в telemetry**: security/compliance риск.
- **Monitoring только инфраструктуры**: CPU нормальный, а checkout не работает.
- **Локальный profiler вместо production observability**: разные задачи.

## Senior-Level Short Answers

**Что такое observability?**

> Способность понять внутреннее состояние системы по telemetry signals: logs, metrics, traces. В отличие от monitoring, observability помогает исследовать неизвестные заранее проблемы.

**Что выбрать: logs, metrics или traces?**

> Метрики - для алертов и трендов, логи - для деталей событий, traces - для пути запроса и latency breakdown. В зрелой системе они связаны через trace/correlation ID.

**Зачем OpenTelemetry, если есть Sentry/Datadog?**

> OpenTelemetry дает vendor-neutral instrumentation и единый формат. Приложение отправляет OTLP в Collector, а Collector уже маршрутизирует в Sentry, Tempo, Datadog или другой backend.

**Как проектировать алерты?**

> От пользовательского impact и SLO: error rate, latency, success rate, queue lag. Алерт должен быть actionable, с owner, dashboard и runbook.

**Как снизить стоимость observability?**

> Sampling, retention policies, агрегация метрик, контроль cardinality, исключение healthcheck/noisy endpoints, Collector-side filtering и scrub.

**Как связать ошибку в Sentry с логами?**

> Добавить `trace_id`/`correlation_id` в Sentry scope/tags и в structured logs. Тогда issue в Sentry ведет к trace и связанным логам.

## Mini-Practice

### Задание 1: API endpoint

Для endpoint `POST /api/orders/{id}/pay` опишите:

- какие метрики нужны;
- какие spans создать;
- какие логи писать;
- какие поля нельзя логировать;
- какой SLO предложить.

Ожидаемый ответ:

- RED metrics по route name `orders.pay`;
- spans: controller, validation, DB transaction, payment provider, event publish;
- logs: payment started/failed/succeeded с `order_id`, `provider`, `correlation_id`, без card/token;
- SLO: например 99.5% successful payment API under 1s excluding provider-declared downtime.

### Задание 2: Очередь

Для async job `SendInvoiceEmail` опишите:

- как пробросить trace context;
- какие метрики нужны;
- что алертить;
- что писать в Sentry.

Ожидаемый ответ:

- сохранить `traceparent`/correlation ID в message headers/payload;
- metrics: queue depth, job duration, retries, failures, age of oldest message;
- alerts: growing lag, high failure rate, exhausted retries;
- Sentry: final failure с invoice/customer safe ID, release, worker name, trace context.

### Задание 3: Инцидент

Ситуация: после релиза выросли 502 на `/checkout`.

Чеклист:

- проверить release marker;
- сравнить error rate до/после;
- открыть Sentry issues по release;
- найти trace для 502;
- проверить upstream spans;
- проверить логи по `trace_id`;
- rollback или feature flag off;
- добавить тест/алерт/instrumentation после postmortem.

## Чеклисты

### Production-ready observability checklist

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

### Interview checklist

- Уметь объяснить observability vs monitoring.
- Уметь выбрать logs/metrics/traces под задачу.
- Уметь объяснить trace/span/context propagation/exporter/collector.
- Знать RED, USE, SLI/SLO/SLA.
- Уметь описать debugging incident flow.
- Назвать риски: cardinality, cost, sampling, PII, alert fatigue.
- Привести PHP/Laravel/Symfony integration points.

## Self-Check Questions

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

- OpenTelemetry Docs: https://opentelemetry.io/docs/
- OpenTelemetry PHP: https://opentelemetry.io/docs/languages/php/
- W3C Trace Context: https://www.w3.org/TR/trace-context/
- Sentry Docs: https://docs.sentry.io/
- Sentry PHP: https://docs.sentry.io/platforms/php/
- Sentry Laravel: https://docs.sentry.io/platforms/php/guides/laravel/
- Sentry Symfony: https://docs.sentry.io/platforms/php/guides/symfony/
- Prometheus Docs: https://prometheus.io/docs/introduction/overview/
- Grafana Docs: https://grafana.com/docs/
- Google SRE Book: https://sre.google/sre-book/table-of-contents/
- Google SRE Workbook: https://sre.google/workbook/table-of-contents/
- Martin Fowler - Observability: https://martinfowler.com/tags/observability.html
- Honeycomb Observability Resources: https://www.honeycomb.io/resources
