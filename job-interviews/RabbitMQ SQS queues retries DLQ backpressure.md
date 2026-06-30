# RabbitMQ/SQS: очереди, retries, DLQ, backpressure

Гайд для подготовки к интервью на позицию Lead/Senior PHP backend developer. Фокус: не только знать термины, а уметь объяснить компромиссы, эксплуатацию и типичные сбои.

## Базовая модель message broker

Message broker отделяет producer от consumer и дает асинхронную обработку, буферизацию пиков, retry, fan-out, интеграцию сервисов и контроль нагрузки.

Ключевые понятия:

- Producer публикует сообщение.
- Broker хранит или маршрутизирует сообщение.
- Consumer/worker получает сообщение и обрабатывает его.
- Queue хранит сообщения до обработки.
- Message содержит payload и metadata: headers, id, correlation id, timestamp, trace id, delivery count.
- Ack подтверждает успешную обработку.
- Retry повторяет обработку после ошибки.
- DLQ/dead-letter queue хранит сообщения, которые нельзя обработать штатно.

Очереди не заменяют транзакции, базу данных и бизнес-инварианты. Они дают eventual consistency, поэтому код должен быть идемпотентным и устойчивым к дублям, задержкам и out-of-order доставке.

## RabbitMQ: exchanges, queues, bindings, routing keys

В RabbitMQ producer обычно публикует не напрямую в queue, а в exchange. Exchange решает, в какие queues отправить сообщение.

Основные типы exchange:

- `direct`: routing key должен совпасть с binding key. Хорошо для команд и явной маршрутизации.
- `topic`: pattern matching по routing key, например `billing.invoice.*` или `*.created`. Хорошо для событийных доменов.
- `fanout`: отправляет во все привязанные queues, routing key игнорируется. Хорошо для broadcast.
- `headers`: маршрутизация по headers, используется реже из-за сложности и стоимости.

Binding связывает exchange и queue. Routing key у producer и binding key у queue определяют маршрут.

Пример рассуждения на интервью:

> Для event-driven интеграции я предпочту `topic exchange` с ключами вроде `order.created.v1`, а для внутренних job-команд можно использовать `direct exchange`. Producer не должен знать конкретные queue names, иначе появляется сильная связность.

## Ack, nack, reject

RabbitMQ доставляет сообщение consumer'у и ждет подтверждения, если включен manual ack.

- `ack`: обработка успешна, сообщение удаляется из queue.
- `nack(requeue=true)`: обработка неуспешна, сообщение возвращается в очередь.
- `nack(requeue=false)`: сообщение не возвращается; при наличии DLX уйдет в dead-letter exchange.
- `reject`: похож на `nack`, но обычно для одного сообщения; `nack` умеет bulk/multiple.

Senior-правило: ack ставится после durable side effect. Если ack до записи в БД или внешнего вызова, возможна потеря работы. Если side effect выполнен, а ack не дошел до broker, будет дубль, поэтому нужна идемпотентность.

Опасность `requeue=true`: можно получить tight retry loop, когда poison message мгновенно возвращается одному из consumer'ов и сжигает CPU.

## Durability и надежность в RabbitMQ

Для снижения риска потери сообщений нужны несколько уровней:

- Durable queue, чтобы queue пережила restart broker.
- Persistent messages, чтобы сообщения записывались на диск.
- Publisher confirms, чтобы producer знал, что broker принял сообщение.
- Consumer manual ack, чтобы broker не удалял сообщение до завершения обработки.
- Quorum queues для большей устойчивости, если требуется репликация и предсказуемое поведение при сбоях.

Важно: durable queue без persistent messages не гарантирует сохранность сообщений. Persistent messages без publisher confirms не дают producer'у уверенности, что публикация состоялась.

## Prefetch и QoS

Prefetch ограничивает количество unacked сообщений на consumer/channel. Это главный инструмент backpressure на стороне RabbitMQ consumer'ов.

Пример:

- `prefetch=1`: worker берет одно сообщение, обрабатывает, ack, берет следующее. Хорошо для тяжелых задач и честного распределения.
- `prefetch=10..100`: выше throughput для быстрых задач, но больше риск неравномерного распределения и роста памяти.
- Слишком большой prefetch: один worker может забрать пачку сообщений и стать bottleneck, пока другие простаивают.

Senior-ответ:

> Prefetch подбирается по времени обработки, памяти, SLA и fairness. Для тяжелых PHP jobs я начну с малого prefetch, измерю latency и utilization, потом увеличу, если broker/DB/API выдерживают нагрузку.

## Retry patterns

Retry нужен только для временных ошибок: network timeout, 5xx внешнего API, deadlock, rate limit. Для бизнес-ошибок retry бесполезен.

Паттерны retry:

- Immediate retry в worker: простой, но может блокировать worker и усиливать перегрузку.
- Delayed retry через отдельные retry queues с TTL и DLX: классический RabbitMQ-подход без busy waiting.
- Exponential backoff: 1m, 5m, 30m, 2h; снижает давление на зависимость.
- Jitter: случайный разброс задержки, чтобы избежать thundering herd.
- Retry budget: ограничение суммарного числа попыток или времени жизни.

Для RabbitMQ часто строят цепочку:

1. Main queue.
2. При ошибке `nack(requeue=false)` или publish в retry exchange.
3. Retry queue с TTL.
4. После TTL сообщение dead-letter'ится обратно в main exchange.
5. После превышения лимита попыток сообщение уходит в DLQ.

Нужно хранить attempt count в headers или использовать framework metadata. Нельзя бесконечно гонять сообщение между main и retry queues.

## DLQ и dead-letter exchange

DLQ нужна не для "починки магией", а для изоляции проблемных сообщений и сохранения evidence.

Сообщение попадает в DLQ, когда:

- Consumer отклонил сообщение без requeue.
- Истек TTL сообщения или очереди.
- Превышен лимит длины queue.
- В SQS превышен `maxReceiveCount` в redrive policy.

Что должно быть в DLQ-сообщении:

- Original payload.
- Headers/correlation id/trace id.
- Error class/message, если добавляет приложение.
- Attempt count.
- Время первой и последней ошибки.
- Версия схемы сообщения.

Операционный процесс:

- Alert при росте DLQ.
- Dashboard с группировкой по error type и message type.
- Runbook: inspect, fix code/data/config, replay small batch, monitor.
- Безопасный replay, который не ломает идемпотентность и не отправляет старые side effects повторно.

## Poison messages

Poison message всегда падает при обработке: плохая схема, отсутствующие данные, невалидный enum, несовместимая версия, баг в consumer.

Опасности:

- Бесконечные retries.
- Блокировка FIFO-группы.
- Маскировка реальной причины за сотнями одинаковых ошибок.
- Перегрузка внешних сервисов повторными вызовами.

Правильное поведение:

- Отличать transient и permanent exceptions.
- Permanent сразу отправлять в DLQ/failure transport.
- Логировать структурно: `message_type`, `message_id`, `attempt`, `exception_class`, `correlation_id`.
- Не логировать секреты из payload.

## Ordering

Очередь не всегда означает строгий порядок обработки.

RabbitMQ:

- В одной queue порядок доставки обычно FIFO, но multiple consumers, requeue, retries и prefetch ломают порядок завершения обработки.
- Для строгого порядка нужен один consumer или partitioning по ключу в отдельные queues/streams, но это снижает throughput.

SQS:

- Standard queue дает at-least-once delivery и best-effort ordering.
- FIFO queue дает порядок внутри `MessageGroupId` и exactly-once processing semantics на уровне дедупликации SQS, но приложение все равно должно быть идемпотентным.

Senior-ответ:

> Я не обещаю глобальный ordering без явной причины. Чаще нужен порядок по aggregate id: order, user, account. Тогда партиционируем по ключу и принимаем компромисс между throughput и consistency.

## Idempotency и deduplication

Брокеры обычно дают at-least-once delivery. Значит, дубль - нормальный сценарий, а не исключение.

Идемпотентность можно обеспечить так:

- Idempotency key/message id в каждом сообщении.
- Таблица `processed_messages` с unique key: consumer name + message id.
- Business unique constraints: например, `payment_id` уникален.
- Upsert вместо blind insert.
- State machine с допустимыми переходами.
- Outbox/inbox pattern для атомарной публикации и приема событий.

Deduplication:

- На producer side: не публиковать один и тот же command повторно без idempotency key.
- На consumer side: проверять message id перед side effect.
- В SQS FIFO: использовать `MessageDeduplicationId` или content-based deduplication, но помнить про ограниченное deduplication window.

Анти-паттерн: считать, что если RabbitMQ/SQS "обычно" не дублирует, то можно не делать идемпотентность.

## Backpressure

Backpressure - механизм, который не дает системе принимать больше работы, чем она может безопасно обработать.

Уровни backpressure:

- Producer throttling: ограничить скорость публикации.
- Broker limits: max queue length, TTL, memory/disk alarms.
- Consumer prefetch/concurrency: не брать больше сообщений, чем можно обработать.
- Application rate limits: ограничить обращения к БД и внешним API.
- Autoscaling: добавлять workers по метрикам, но только если downstream выдерживает.
- Circuit breaker: временно прекращать попытки при деградации зависимости.
- Load shedding: явно отклонять или переносить низкоприоритетные задачи.

Метрики для решений:

- Queue depth.
- Message age / oldest message age.
- Publish rate и consume/ack rate.
- Redelivery rate.
- Retry rate.
- DLQ growth.
- Worker processing time p50/p95/p99.
- DB/API latency и error rate.

Senior-ответ:

> Если очередь растет, я сначала смотрю не только на количество сообщений, а на age, ack rate, ошибки и downstream. Просто добавить workers может добить базу или внешний API.

## Amazon SQS: важные особенности

SQS - managed queue service. Нет exchanges как в RabbitMQ. Producer отправляет в queue, consumer poll'ит queue.

### Visibility timeout

Когда consumer получил сообщение, SQS не удаляет его, а скрывает на время `visibility timeout`. Если consumer удалил сообщение через `DeleteMessage`, оно считается обработанным. Если не удалил, после timeout сообщение снова станет видимым.

Риски:

- Timeout меньше времени обработки: один и тот же job параллельно обработают два worker'а.
- Timeout слишком большой: при падении worker'а retry будет ждать слишком долго.
- Для долгих задач нужно продлевать visibility timeout или дробить задачу.

Правило:

> Visibility timeout должен быть больше p99 времени обработки плюс запас, но не настолько большим, чтобы recovery после сбоя занимал часы.

### Standard vs FIFO

Standard queue:

- Очень высокий throughput.
- At-least-once delivery.
- Best-effort ordering.
- Возможны дубли.

FIFO queue:

- Порядок внутри `MessageGroupId`.
- Deduplication через `MessageDeduplicationId` или content-based deduplication.
- Ниже throughput и больше ограничений.
- Poison message может блокировать группу сообщений.

Выбор:

- Standard для независимых фоновых задач, email, webhooks, image processing.
- FIFO для операций, где важен порядок по сущности: ledger/account/order lifecycle.

### Long polling

Long polling уменьшает пустые ответы и стоимость polling'а. Consumer задает `WaitTimeSeconds` до 20 секунд. Для production почти всегда лучше long polling, чем tight loop с short polling.

### SQS DLQ

DLQ настраивается через redrive policy:

- Source queue указывает DLQ.
- `maxReceiveCount` определяет, после скольких получений без удаления сообщение уйдет в DLQ.
- Для FIFO DLQ надо учитывать, что перенос сообщения может нарушить ожидания по порядку бизнес-процесса.

## Worker scaling

Масштабирование worker'ов должно учитывать bottleneck всей цепочки.

Что смотреть перед увеличением concurrency:

- CPU/RAM worker'ов.
- DB connections pool.
- Locks и deadlocks.
- Rate limits внешних API.
- Average и p99 job duration.
- Queue age, не только queue length.
- Количество retries и redeliveries.

Подход:

- Горизонтально масштабировать stateless workers.
- Ограничивать concurrency на тип job'а.
- Разделять очереди по приоритету и resource profile.
- Использовать отдельные workers для slow/heavy jobs.
- Делать graceful shutdown: остановить прием новых сообщений, дождаться текущей обработки, ack/nack корректно.

## Monitoring и alerting

Минимальный production-набор:

- Queue depth и oldest message age.
- Publish/consume/ack rates.
- Error/retry/DLQ rates.
- Redelivered messages в RabbitMQ.
- Unacked messages в RabbitMQ.
- SQS `ApproximateAgeOfOldestMessage`, `ApproximateNumberOfMessagesVisible`, `ApproximateNumberOfMessagesNotVisible`.
- Worker restarts, memory leaks, OOM, exit codes.
- Time-to-process и end-to-end latency от publish до ack.

Alert'ы лучше строить по impact:

- Oldest message age превышает SLA.
- DLQ растет.
- Consume rate сильно ниже publish rate длительное время.
- Redelivery/retry spike.
- Workers живы, но ack rate около нуля.

## Laravel queues

В Laravel важные элементы:

- Job реализует `ShouldQueue`.
- `tries`, `backoff`, `retryUntil`, `timeout` можно задавать на job или worker level.
- Worker options: `php artisan queue:work --tries=3 --backoff=3 --timeout=60`.
- Failed jobs просматриваются через `queue:failed`, повторяются через `queue:retry`, чистятся через `queue:flush`/`queue:forget`.
- Метод `failed(?Throwable $exception)` подходит для компенсаций, уведомлений и observability.
- Для SQS driver важно согласовать Laravel timeout и SQS visibility timeout.
- Horizon полезен для Redis queues: метрики, балансировка, supervision; для SQS/RabbitMQ часто нужны внешние метрики и supervisor/systemd/Kubernetes.

Pitfall в Laravel: `--timeout` worker'а должен быть меньше `retry_after`/visibility timeout. Иначе job может стать видимым и выполниться другим worker'ом, пока первый еще работает.

## Symfony Messenger

В Symfony Messenger важные элементы:

- Message - DTO команды/события.
- Handler обрабатывает message.
- Transport определяет, куда отправлять и откуда получать.
- Routing связывает message class с transport.
- Retry strategy настраивает `max_retries`, `delay`, `multiplier`, `max_delay`, jitter в новых версиях.
- Failure transport хранит сообщения после исчерпания retries.
- `UnrecoverableExceptionInterface` позволяет не retry'ить permanent errors.
- AMQP transport поддерживает exchange/queues/binding keys через options.
- SQS transport доступен через Symfony Amazon SQS Messenger bridge.

Команды, которые стоит знать:

```bash
php bin/console messenger:consume async -vv
php bin/console messenger:failed:show
php bin/console messenger:failed:retry
php bin/console messenger:failed:remove
```

Senior-заметка: Messenger abstractions удобны, но на интервью важно показать, что вы понимаете underlying broker semantics: ack, visibility timeout, DLQ, prefetch, retry loops.

## Частые pitfalls

- Ack до фактического завершения side effect.
- Отсутствие идемпотентности при at-least-once delivery.
- Бесконечный retry без лимита и DLQ.
- Immediate retry при outage внешнего API.
- Слишком большой prefetch для тяжелых jobs.
- Один общий queue для быстрых и долгих задач.
- Нет message schema versioning.
- Нет correlation id и traceability.
- DLQ есть, но никто ее не мониторит.
- Replay из DLQ запускается пачкой без rate limit.
- Laravel/SQS timeout mismatch.
- Visibility timeout меньше реального времени обработки.
- FIFO используется ради "надежности", хотя нужен throughput.
- Масштабирование workers без проверки DB/API bottleneck.
- Секреты и персональные данные в payload/logs.

## Senior answers

### Как спроектировать retry?

> Сначала классифицирую ошибки: transient retry'им с exponential backoff и jitter, permanent сразу в DLQ/failure transport. Ограничиваю число попыток и общее время жизни. Для RabbitMQ использую retry queues с TTL и DLX или delayed exchange, для SQS - visibility timeout плюс redrive policy. Все handlers идемпотентны.

### Что делать, если queue depth растет?

> Смотрю oldest message age, publish/ack rate, error rate, retry rate, unacked/not visible messages и downstream latency. Если consumer'ы не успевают из-за CPU - масштабирую. Если bottleneck в БД/API - ограничиваю concurrency, включаю backoff/circuit breaker и разделяю очереди.

### Как избежать дублей?

> Не пытаюсь полностью избежать на уровне broker. Проектирую consumer как идемпотентный: message id, unique constraints, processed_messages/inbox, state transitions. Ack делаю после durable side effect.

### Когда RabbitMQ, когда SQS?

> RabbitMQ хорош, когда нужна гибкая маршрутизация через exchanges, low latency, control над topology, on-prem/self-managed или сложные routing patterns. SQS хорош, когда нужен managed сервис, простая очередь, высокая доступность без эксплуатации broker и интеграция с AWS. Компромиссы SQS - polling, visibility timeout, меньше routing primitives.

### Нужна ли строгая очередность?

> Глобальная очередность дорогая и часто не нужна. Обычно нужен порядок по aggregate id. Тогда в SQS FIFO использую MessageGroupId, а в RabbitMQ - partitioning/consistent routing или один consumer на ключ/queue, понимая цену в throughput.

## Self-check

- Могу ли я объяснить разницу между exchange и queue?
- Чем `nack(requeue=true)` опасен при poison message?
- Почему ack должен быть после side effect?
- Что произойдет в SQS, если visibility timeout меньше времени обработки?
- Чем Standard SQS отличается от FIFO?
- Как спроектировать retry с exponential backoff в RabbitMQ?
- Какие метрики покажут, что workers не справляются?
- Почему DLQ без runbook бесполезна?
- Как сделать consumer идемпотентным?
- Почему увеличение workers может ухудшить ситуацию?

## Mini-practice

1. Нарисуйте схему RabbitMQ для `order.created`, где billing, email и analytics получают событие независимо.
2. Опишите retry flow для внешнего API, который иногда отвечает 503.
3. Спроектируйте таблицу `processed_messages` для идемпотентного consumer'а в PHP-сервисе.
4. Настройте SQS-подход для job, который обычно идет 20 секунд, но p99 равен 2 минутам.
5. Разберите инцидент: queue depth растет, CPU workers 20%, DB latency p99 выросла до 5 секунд.
6. Опишите безопасный replay 10 000 сообщений из DLQ.

## Checklist перед production

- Есть manual ack/delete после успешной обработки.
- Handlers идемпотентны.
- Есть лимит retries и DLQ/failure transport.
- Retry использует backoff и jitter.
- Permanent errors не retry'ятся бесконечно.
- Настроены prefetch/concurrency/timeouts.
- SQS visibility timeout согласован с worker timeout.
- Очереди разделены по приоритету и resource profile.
- Есть monitoring queue age, depth, retries, DLQ, worker health.
- Есть alert'ы и runbook для DLQ.
- Есть correlation id, structured logs и tracing.
- Payload имеет версию схемы.
- Replay из DLQ rate-limited и проверен на staging/small batch.

## Ссылки

- RabbitMQ Documentation: https://www.rabbitmq.com/docs
- RabbitMQ Exchanges: https://www.rabbitmq.com/docs/exchanges
- RabbitMQ Consumers, acknowledgements and prefetch: https://www.rabbitmq.com/docs/consumers
- RabbitMQ Dead Letter Exchanges: https://www.rabbitmq.com/docs/dlx
- RabbitMQ Reliability Guide: https://www.rabbitmq.com/docs/reliability
- Amazon SQS Developer Guide: https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/welcome.html
- SQS Visibility Timeout: https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-visibility-timeout.html
- SQS Dead-letter Queues: https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html
- SQS FIFO Queues: https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-fifo-queues.html
- SQS Long Polling: https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-short-and-long-polling.html
- Symfony Messenger: https://symfony.com/doc/current/messenger.html
- Symfony Amazon SQS Messenger: https://symfony.com/doc/current/messenger.html#amazon-sqs
- Laravel Queues: https://laravel.com/docs/queues
- Laravel Horizon: https://laravel.com/docs/horizon
