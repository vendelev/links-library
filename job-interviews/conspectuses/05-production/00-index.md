# Production: observability, debugging, delivery, storage

Раздел объединяет production-темы для подготовки к интервью Lead/Senior PHP backend developer. Материал перегруппирован из старых конспектов так, чтобы сквозные темы не повторялись в каждом файле.

## Как читать

1. Начать с incident playbook: общая модель диагностики и mitigation.
2. Затем пройти observability: какие сигналы должны помочь при инциденте.
3. После этого читать HTTP/Nginx и Linux debugging: как искать причину на сетевом и системном уровнях.
4. Отдельно пройти CI/CD: как выкатывать, наблюдать deploy и откатываться.
5. В конце пройти object storage/CDN/media: файлы, S3, MinIO, CDN и большие uploads.

## Файлы

- [01 Observability Sentry OpenTelemetry](01-observability-sentry-opentelemetry.md)
- [02 HTTP DNS TLS Nginx](02-http-dns-tls-nginx.md)
- [03 Linux production debugging](03-linux-production-debugging.md)
- [04 CI/CD Docker GitLab deploy rollback](04-ci-cd-docker-gitlab-deploy-rollback.md)
- [05 Object Storage S3 MinIO CDN media](05-object-storage-s3-minio-cdn-media.md)
- [06 Production incident playbook](06-production-incident-playbook.md)
- [07 Timeouts retries idempotency backpressure](07-timeouts-retries-idempotency-backpressure.md)
- [99 Coverage map](99-coverage-map.md)

## Сквозные связи

- 502/504 в Nginx: [02](02-http-dns-tls-nginx.md), системная диагностика PHP-FPM: [03](03-linux-production-debugging.md), общий incident flow: [06](06-production-incident-playbook.md).
- Timeouts/retries/idempotency: общий материал в [07](07-timeouts-retries-idempotency-backpressure.md), HTTP-слой в [02](02-http-dns-tls-nginx.md), большие uploads в [05](05-object-storage-s3-minio-cdn-media.md).
- Deploy observability и rollback: сигналы в [01](01-observability-sentry-opentelemetry.md), процесс deploy в [04](04-ci-cd-docker-gitlab-deploy-rollback.md), решение во время инцидента в [06](06-production-incident-playbook.md).
- CDN/cache: HTTP caching semantics в [02](02-http-dns-tls-nginx.md), CDN над S3 и immutable keys в [05](05-object-storage-s3-minio-cdn-media.md).
