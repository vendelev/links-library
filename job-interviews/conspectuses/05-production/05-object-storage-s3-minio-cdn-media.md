# Object Storage, S3, MinIO, CDN, media files

Конспект о проектировании хранения файлов: S3-compatible object storage, bucket/key schema, consistency, public/private access, presigned URLs, multipart upload, lifecycle, versioning, encryption, CDN, media processing, MinIO, backup/DR, стоимость и PHP integration.

## Что такое object storage

Object Storage хранит данные как объекты, а не как файлы в POSIX-файловой системе и не как блоки на диске.

Объект состоит из:

- `key`: уникальное имя объекта внутри bucket;
- `value/body`: бинарное содержимое;
- `metadata`: системные и пользовательские метаданные;
- `version id`: если включено versioning;
- ACL/policy-related attributes: права доступа, owner, encryption state.

Ключевая идея: приложение обращается к объекту по API, чаще всего HTTP/S3 API, а не через локальный путь на диске.

Подходит для:

- пользовательских загрузок;
- изображений, видео, документов;
- бэкапов и архивов;
- логов и data lake;
- статических ассетов;
- результатов batch/media processing.

Плохо подходит для:

- частых мелких random-write операций;
- транзакционного обновления части файла;
- POSIX-семантики lock/rename/fsync;
- данных, которые должны часто изменяться in-place.

Senior-ответ:

> Object Storage стоит рассматривать как append/replace-oriented удаленное хранилище объектов с HTTP API, высокой durability, горизонтальным масштабированием и другой моделью консистентности/стоимости по сравнению с локальной файловой системой.

## S3 как де-факто стандарт

Amazon S3 стал стандартом API для object storage. S3-compatible системы: MinIO, Ceph RGW, Wasabi, Backblaze B2 S3 API, DigitalOcean Spaces, Cloudflare R2.

Сущности S3:

- Bucket: контейнер для объектов;
- Key: имя объекта внутри bucket;
- Object: содержимое плюс metadata;
- Region: регион bucket;
- Storage Class: класс хранения с разной ценой и SLA;
- IAM Policy/Bucket Policy: правила доступа;
- Presigned URL: временная подписанная ссылка;
- Multipart Upload: загрузка большого объекта частями;
- Lifecycle Policy: автоматический переход/удаление объектов;
- Versioning: хранение нескольких версий объекта;
- Replication: репликация между bucket/region.

Важно: S3 bucket namespace глобален в рамках AWS partition, поэтому имя bucket должно быть уникальным.

## Bucket, key, prefix, metadata

### Bucket

Bucket обычно проектируют по границам безопасности, окружения и жизненного цикла данных.

Примеры:

- `myapp-prod-private`;
- `myapp-prod-public-assets`;
- `myapp-prod-backups`;
- `myapp-staging-media`.

Не стоит создавать bucket на каждого пользователя без сильной причины. Обычно используют один или несколько bucket и разделяют объекты prefix-ами.

### Key

S3 key - это строка. Символ `/` не создает директории, а только формирует prefix.

```text
users/123/avatar/original.jpg
users/123/avatar/256x256.webp
tenants/acme/documents/2026/06/invoice-789.pdf
uploads/tmp/01J2ABCDEF/file.bin
```

Практики:

- не использовать оригинальное имя файла как единственный идентификатор;
- хранить расширение, если оно полезно для клиента/CDN/content-type;
- избегать PII в key: email, phone, passport, full name;
- использовать UUID/ULID/content hash для уникальности;
- разделять временные и постоянные файлы prefix-ами;
- закладывать tenant/user/project prefix для lifecycle и аудита.

Плохой key:

```text
uploads/ivan.petrov@example.com/passport.pdf
```

Лучше:

```text
tenants/01H.../documents/01J.../original.pdf
```

### Metadata

S3 поддерживает:

- system metadata: `Content-Type`, `Content-Length`, `ETag`, `Last-Modified`, `Cache-Control`, `Content-Disposition`;
- user metadata: `x-amz-meta-*`.

Что часто кладут в metadata:

- MIME type;
- original filename;
- uploader id;
- checksum;
- processing status;
- trace/correlation id.

Metadata не заменяет БД. Бизнес-сущности лучше хранить в PostgreSQL/MySQL, а S3 использовать как blob storage.

Senior-ответ:

> S3 metadata полезна для HTTP-доставки и технической информации, но бизнес-метаданные я держу в БД, потому что по S3 metadata нельзя нормально строить транзакционные запросы, индексы и бизнес-инварианты.

## Консистентность S3

Современный Amazon S3 предоставляет strong read-after-write consistency для `PUT`, `GET`, `LIST`, изменения tags, ACL и metadata.

Это означает:

- после успешного `PUT` новый объект сразу виден при `GET`;
- после overwrite последующий `GET` возвращает новую версию;
- после delete объект сразу не должен возвращаться;
- `LIST` отражает актуальное состояние.

Исторически S3 был eventually consistent для некоторых операций, поэтому старые статьи и ответы могут быть устаревшими.

Нюансы:

- strong consistency S3 не превращает набор объектов в транзакционную БД;
- нет multi-object transaction;
- нет atomic rename директории;
- overwrite одного key опасен при CDN/cache;
- внешние системы вокруг S3 могут иметь собственную задержку: CDN, очереди, индексы, search, БД.

Senior-ответ:

> S3 сейчас strong consistent на уровне отдельных операций с объектами и LIST, но workflow лучше проектировать так, чтобы состояние процесса фиксировалось в БД/очереди, а не выводилось только из наличия набора объектов.

## Public и private files

Public object может читаться без авторизации.

Подходит для:

- публичных картинок;
- статических ассетов;
- публичных документов;
- CDN-distributed контента.

Риски:

- случайная публикация приватных данных;
- невозможность надежно отозвать уже скачанный файл;
- индексация поисковиками;
- hotlinking;
- рост CDN/S3 egress costs.

Private object доступен только через backend, presigned URL, signed CDN URL/cookie или авторизованный proxy.

Подходит для:

- документов пользователей;
- медиа в закрытом SaaS;
- медицинских/финансовых файлов;
- temporary uploads;
- файлов до модерации/virus scan.

Варианты доступа:

- backend скачивает объект и стримит клиенту;
- backend выдает S3 presigned URL;
- backend выдает CloudFront signed URL/cookie;
- CDN обращается к private origin через Origin Access Control/Identity.

Senior-ответ:

> По умолчанию bucket должен быть private и иметь Block Public Access. Public-доступ включается осознанно через отдельный bucket/prefix или через CDN с контролируемой политикой.

## Presigned URLs

Presigned URL - URL с подписью, который временно разрешает конкретную операцию: `GET`, `PUT`, multipart-related операции.

Используется для:

- прямой загрузки файла из браузера/mobile app в S3;
- временной выдачи private file;
- снижения нагрузки на backend;
- загрузки больших файлов без проксирования через PHP.

Плюсы:

- backend не держит большой upload/download поток;
- можно ограничить TTL;
- можно подписать конкретный bucket/key/method;
- хорошо масштабируется.

Минусы и риски:

- URL является bearer token;
- сложно отозвать отдельный presigned URL до истечения TTL;
- нужно валидировать MIME, размер, owner и expected key до выдачи URL;
- после прямой загрузки нужен callback/polling/event, чтобы подтвердить факт загрузки и запустить обработку.

Upload flow:

1. Клиент запрашивает разрешение на загрузку.
2. Backend проверяет пользователя, лимиты, MIME, размер, бизнес-правила.
3. Backend создает запись `uploads` со статусом `pending`.
4. Backend выдает presigned `PUT` или multipart init data.
5. Клиент загружает файл напрямую в S3/MinIO.
6. Клиент сообщает backend об окончании или backend получает S3 event.
7. Backend проверяет объект: exists, size, checksum, content-type.
8. Backend запускает virus scan/media processing.
9. После успешной обработки файл становится доступным.

Senior-ответ:

> Presigned URL не заменяет авторизацию. Авторизация происходит до выдачи ссылки, а после upload я не доверяю клиентскому заявлению и перепроверяю объект в storage.

## Multipart upload

Multipart upload позволяет загружать большой объект частями.

Нужен для:

- файлов от сотен мегабайт;
- нестабильных мобильных/браузерных сетей;
- видео;
- архивов;
- resumable upload сценариев.

Преимущества:

- параллельная загрузка частей;
- retry отдельной part;
- меньше риск потерять весь upload;
- можно возобновлять загрузку на уровне приложения.

Нюансы:

- незавершенные multipart uploads занимают место и стоят денег;
- нужен lifecycle rule для abort incomplete multipart uploads;
- итоговый `ETag` для multipart обычно не простой MD5 всего файла;
- порядок part важен при complete;
- backend должен контролировать upload id, key, owner, part size, лимиты.

Лимиты S3:

- максимум 10 000 parts;
- минимальный размер part обычно 5 MiB, кроме последней;
- максимальный размер объекта 5 TiB.

Senior-ответ:

> Для больших uploads я использую multipart, храню состояние загрузки в БД и обязательно настраиваю lifecycle abort для incomplete multipart, иначе можно незаметно платить за мусор.

## Lifecycle policies и versioning

Lifecycle policy автоматизирует управление объектами.

Правила:

- удалить temporary uploads старше 24 часов;
- abort incomplete multipart uploads через 1-7 дней;
- переместить старые логи в Glacier/Archive;
- удалить noncurrent versions через 30 дней;
- удалить thumbnails при истечении срока хранения оригинала;
- очистить quarantine files после N дней.

Плюсы:

- снижение стоимости;
- меньше ручной уборки;
- enforce retention policy;
- управление временными файлами.

Риски:

- случайное удаление нужных данных;
- несовпадение lifecycle с business retention;
- восстановление из archive может занимать часы;
- lifecycle не мгновенный, не стоит использовать как realtime scheduler.

Versioning хранит несколько версий объекта при overwrite/delete.

Плюсы:

- защита от случайного удаления;
- восстановление после ошибочного overwrite;
- база для Object Lock/retention;
- полезно для audit-sensitive данных.

Минусы:

- растет storage cost;
- delete создает delete marker, а не обязательно удаляет байты;
- lifecycle для noncurrent versions обязателен;
- приложение должно понимать latest или конкретный version id.

Senior-ответ:

> Versioning полезен как safety net, но без lifecycle на noncurrent versions он может резко увеличить storage cost. Для бизнес-версий документов я чаще храню явную модель версий в БД и key на immutable object.

## Encryption, policies, security

Варианты encryption:

- SSE-S3: server-side encryption with S3-managed keys;
- SSE-KMS: server-side encryption with AWS KMS keys;
- SSE-C: server-side encryption with customer-provided keys;
- client-side encryption: шифрование до отправки в storage.

Что выбрать:

- SSE-S3: default baseline;
- SSE-KMS: контроль ключей, audit, rotation, separation of duties;
- client-side: storage provider не должен видеть plaintext, но сложнее key management.

SSE-KMS нюансы:

- дополнительные costs за KMS API calls;
- throttling/limits KMS;
- IAM должен разрешать `kms:Decrypt`, `kms:Encrypt`, `kms:GenerateDataKey`;
- CDN/origin access должен учитывать KMS permissions.

Access mechanisms:

- IAM user/role policy;
- bucket policy;
- bucket public access block;
- ACL, желательно избегать в новых схемах;
- Object Ownership / bucket-owner-enforced;
- VPC endpoint policy;
- KMS key policy;
- CloudFront OAC/OAI для private origin.

Принципы:

- least privilege;
- отдельные роли для read/write/admin;
- отдельные credentials на окружение;
- запрет public access по умолчанию;
- запрет wildcard `s3:*` на все bucket;
- ограничение по prefix, если возможно;
- audit через CloudTrail/server access logs/S3 Inventory;
- secrets не хранить в коде и `.env` репозитория.

Плохая policy:

```json
{
  "Action": "s3:*",
  "Resource": "*",
  "Effect": "Allow"
}
```

Лучше:

```json
{
  "Action": ["s3:GetObject", "s3:PutObject", "s3:DeleteObject"],
  "Resource": "arn:aws:s3:::myapp-prod-private/tenants/*",
  "Effect": "Allow"
}
```

Senior-ответ:

> Я не полагаюсь на obscurity key. Если файл приватный, это enforce'ится policy/IAM/CDN signing, а не просто сложным URL.

## CDN и object storage

CDN ставится перед object storage для:

- снижения latency;
- уменьшения нагрузки на origin;
- снижения egress из origin;
- TLS/HTTP/2/HTTP/3 edge termination;
- географической доставки;
- edge caching;
- signed URLs/cookies;
- image optimization, если CDN это поддерживает.

```text
Client -> CDN -> S3/MinIO origin
Client -> CloudFront -> S3 bucket with OAC
```

### Cache-Control и immutable keys

Главный инструмент управления CDN cache - HTTP headers:

```text
Cache-Control: public, max-age=31536000, immutable
```

Для immutable ассетов с content hash в имени можно ставить долгий TTL. Для mutable key лучше короткий TTL или versioned URL.

Плохой подход:

```text
/users/123/avatar.jpg
```

Лучше:

```text
/users/123/avatar/01JABC256.webp
```

И в БД хранить текущий active key.

Invalidation очищает объект из CDN cache.

Минусы:

- может стоить денег;
- не всегда мгновенный;
- wildcard invalidation надо использовать аккуратно;
- при высокой частоте обновлений лучше versioned keys.

Senior-ответ:

> Я предпочитаю immutable object keys и cache forever, а не постоянные invalidation. Invalidation оставляю для редких emergency/update сценариев.

## Signed CDN URLs и cookies

Для private media через CDN лучше использовать CDN-level authorization:

- CloudFront Signed URL;
- CloudFront Signed Cookies;
- token authentication у CDN provider;
- origin private, доступен только CDN.

Signed URL подходит для:

- доступа к одному файлу;
- короткоживущей ссылки;
- download link.

Signed Cookies подходят для:

- доступа к набору файлов;
- HLS/DASH видео с множеством сегментов;
- галереи/альбома;
- private frontend area.

Senior-ответ:

> Для приватного видео через CDN я не буду выдавать presigned S3 URL на каждый сегмент. Лучше закрыть origin и использовать signed cookies/URL на уровне CDN.

## Media processing pipeline

Pipeline пользовательского медиа:

1. Upload в `tmp/` или `incoming/` prefix.
2. Создание записи в БД со статусом `uploaded`/`pending_scan`.
3. Virus scan.
4. MIME sniffing и validation реального содержимого.
5. Извлечение metadata: размер изображения, duration видео, pages count.
6. Генерация derivatives: thumbnails, previews, transcoded video, waveform.
7. Сохранение derivatives в отдельные keys.
8. Обновление БД в транзакции: `ready` или `failed`.
9. Очистка временных объектов.
10. Публикация события для downstream-систем.

```text
Browser -> S3 tmp -> Event/Queue -> Worker -> Scan -> Process -> S3 final -> DB ready -> CDN
```

В PHP heavy media processing часто выносят в workers:

- Laravel Queue / Horizon;
- Symfony Messenger;
- отдельный Go/Python/Node service;
- FFmpeg workers;
- serverless functions, если подходит по лимитам.

Senior-ответ:

> Загрузку файла и готовность файла я разделяю. Факт upload не означает, что файл можно показывать пользователям: он должен пройти validation, scan и processing.

## Thumbnails, derivatives, virus scanning

Подходы к derivatives:

- eager generation: создать thumbnails сразу после upload;
- lazy generation: создать при первом запросе;
- on-the-fly CDN/image proxy: трансформация на edge/proxy;
- hybrid: популярные размеры заранее, редкие по запросу.

Учитывать:

- original всегда хранить отдельно;
- derivative keys должны быть детерминированными;
- в key включать размер/формат/версию алгоритма;
- при изменении алгоритма использовать новый prefix/version;
- не доверять extension, проверять реальный MIME;
- защищаться от image bombs и слишком больших dimensions.

```text
media/01JABC/original
media/01JABC/derivatives/v2/256x256.webp
media/01JABC/derivatives/v2/1024x768.webp
```

Virus scanning:

- ClamAV worker;
- managed malware scanning service;
- antivirus gateway;
- sandbox для подозрительных файлов;
- асинхронный scan через queue.

Практики:

- файл до scan хранить в private quarantine prefix;
- не отдавать файл пользователю до статуса `clean`;
- ограничить размер и типы файлов до upload;
- логировать scan result;
- при `infected` удалять/изолировать объект и сохранять audit event;
- учитывать zip bombs, nested archives, timeout.

Senior-ответ:

> Virus scan должен быть частью state machine, а не best-effort side task. До статуса `clean` файл не должен попадать в public/CDN namespace.

## Large uploads

Проблемы больших uploads:

- timeout PHP-FPM/nginx;
- memory usage при проксировании через backend;
- нестабильная сеть;
- повторная загрузка с нуля;
- лимиты reverse proxy;
- стоимость незавершенных uploads;
- медленная post-processing pipeline.

Решение:

- direct-to-S3 upload через presigned URLs;
- multipart upload;
- state в БД;
- checksum validation;
- background processing;
- progress на клиенте;
- lifecycle abort incomplete multipart;
- rate limits и квоты.

Для tus/resumable uploads можно использовать отдельный upload service или библиотеку, но финальное хранилище часто остается S3-compatible.

Senior-ответ:

> Я стараюсь не гонять гигабайтные uploads через PHP. PHP должен авторизовать и оркестрировать, а data plane лучше отдавать S3/CDN/upload service.

## MinIO

MinIO - S3-compatible object storage, часто используется:

- локально для разработки;
- on-premise;
- private cloud;
- Kubernetes deployments;
- edge environments;
- как альтернатива managed S3 при требованиях к контролю инфраструктуры.

Плюсы:

- совместимость с S3 API;
- удобно для dev/test;
- self-hosted контроль;
- высокая производительность при правильной настройке;
- erasure coding, versioning, policies, replication.

Минусы:

- эксплуатация на вашей стороне;
- нужно проектировать disks/nodes/network;
- backup/DR/SLA не появляются автоматически;
- совместимость с S3 не всегда означает 100% идентичную семантику всех AWS-фич;
- обновления, мониторинг, capacity planning - ваша ответственность.

Local dev endpoint:

```text
http://minio:9000
http://minio:9000/bucket/key
```

Senior-ответ:

> MinIO хорош как S3-compatible слой, но я не называю его просто локальным S3. В production это отдельная storage-система, требующая полноценной эксплуатации, мониторинга, capacity planning и DR.

## Backups и disaster recovery

Object storage часто имеет высокую durability, но это не то же самое, что backup.

Что может пойти не так:

- баг приложения удалил все объекты;
- credentials скомпрометированы;
- lifecycle rule удалил нужные данные;
- ransomware зашифровал и перезаписал объекты;
- удален bucket;
- ошибка оператора;
- региональная авария;
- corruption на уровне приложения.

Защита:

- versioning;
- MFA delete/Object Lock, если применимо;
- cross-region replication;
- separate backup account/project;
- least privilege credentials;
- lifecycle для old versions;
- регулярные restore drills;
- inventory и reconciliation с БД;
- immutable backups для критичных данных.

Senior-ответ:

> Durability S3 защищает от потери дисков у провайдера, но не от логического удаления вашим приложением. Для этого нужны versioning, backup strategy, access isolation и регулярная проверка восстановления.

## Стоимость

Cost drivers:

- хранение GB/month;
- requests: `PUT`, `GET`, `LIST`, lifecycle transitions;
- data transfer out / egress;
- CDN traffic;
- invalidation;
- storage class transitions;
- retrieval из archive классов;
- KMS requests;
- replication traffic;
- logging/inventory/analytics;
- незавершенные multipart uploads;
- noncurrent versions.

Оптимизации:

- CDN cache с правильным `Cache-Control`;
- immutable keys вместо invalidation;
- lifecycle для temporary и old data;
- cleanup incomplete multipart;
- правильные storage classes;
- compression/transcoding;
- thumbnails нужных размеров, а не бесконечная матрица;
- избегать частого `LIST` в runtime;
- batch jobs вместо N+1 object requests;
- мониторинг egress и hot objects.

Senior-ответ:

> В object storage дорого может быть не только хранение, но и запросы, egress, KMS и мусор от incomplete multipart/old versions. Поэтому storage design должен включать cost model.

## PHP integration

### AWS SDK for PHP

Операции:

- `putObject`;
- `getObject`;
- `headObject`;
- `deleteObject`;
- `createPresignedRequest`;
- multipart upload helpers.

Практики:

- использовать IAM role в AWS, а не static keys;
- для local/dev MinIO использовать custom endpoint;
- не читать большие файлы целиком в память;
- использовать streams;
- retry/backoff для transient errors;
- логировать request id при ошибках;
- различать `404 Not Found`, `403 AccessDenied`, network timeout.

### Laravel

Laravel использует Flysystem.

```php
// config/filesystems.php
's3' => [
    'driver' => 's3',
    'key' => env('AWS_ACCESS_KEY_ID'),
    'secret' => env('AWS_SECRET_ACCESS_KEY'),
    'region' => env('AWS_DEFAULT_REGION'),
    'bucket' => env('AWS_BUCKET'),
    'url' => env('AWS_URL'),
    'endpoint' => env('AWS_ENDPOINT'),
    'use_path_style_endpoint' => env('AWS_USE_PATH_STYLE_ENDPOINT', false),
],
```

```php
Storage::disk('s3')->put($key, $stream);
Storage::disk('s3')->temporaryUrl($key, now()->addMinutes(10));
Storage::disk('s3')->exists($key);
Storage::disk('s3')->delete($key);
```

Нюансы:

- `visibility => public` не должен случайно включаться для private данных;
- `temporaryUrl` зависит от adapter capabilities;
- для MinIO часто нужен `AWS_ENDPOINT` и `AWS_USE_PATH_STYLE_ENDPOINT=true`;
- queue jobs должны быть идемпотентны;
- heavy processing не выполнять в request lifecycle;
- model должен хранить key, disk, metadata, status, size, checksum.

### Symfony

Часто используют:

- FlysystemBundle;
- AWS SDK напрямую;
- Symfony Messenger для async processing;
- VichUploaderBundle, если нужен higher-level upload workflow.

Практики:

- storage abstraction через service;
- domain не должен зависеть напрямую от SDK;
- streaming responses для больших downloads;
- signed URL generation в application service;
- async jobs для thumbnails/scan;
- transactional boundary: БД фиксирует состояние, object storage хранит bytes.

Senior-ответ:

> В PHP я разделяю control plane и data plane: backend принимает решение, проверяет права и меняет состояние в БД, а большие байты идут напрямую между клиентом и object storage/CDN.

## Типичные pitfalls

- Хранить приватные файлы в public bucket.
- Использовать оригинальное имя файла как key.
- Доверять `Content-Type` от клиента.
- Показывать файл до virus scan.
- Проксировать большие uploads через PHP-FPM.
- Не настраивать abort incomplete multipart uploads.
- Перезаписывать один и тот же key за CDN с долгим TTL.
- Делать `LIST` в hot path вместо хранения key в БД.
- Считать S3 backup-ом без versioning/replication/restore drill.
- Не учитывать egress/CDN/KMS/request costs.
- Хранить PII в key или public metadata.
- Выдавать presigned URL на слишком долгий срок.
- Не проверять owner при завершении upload.
- Смешивать tmp, quarantine и public final files в одном prefix без state machine.
- Не делать idempotency для processing jobs.
- Не ограничивать размер и dimensions изображений.
- Считать MinIO полностью идентичным AWS S3 по всем edge cases.

## Senior answers

**Как организовать загрузку больших файлов?**

> Backend выдает presigned multipart upload после проверки прав и лимитов, хранит upload state в БД, клиент грузит parts напрямую в S3, затем backend подтверждает объект через `HEAD`, проверяет size/checksum и запускает async scan/processing. Incomplete multipart чистится lifecycle policy.

**Как сделать приватные файлы через CDN?**

> Bucket остается private, CloudFront получает доступ к origin через OAC, пользователю выдаются signed URL или signed cookies. Для HLS лучше signed cookies, потому что сегментов много. Origin напрямую из интернета недоступен.

**Как избежать CDN cache при обновлении аватара?**

> Не перезаписывать `avatar.jpg`, а писать новый immutable key с версией/ULID/hash, обновлять ссылку в БД и отдавать долгий `Cache-Control`. Invalidation использовать только как fallback.

**Где хранить metadata файла?**

> Технические HTTP metadata можно хранить в object metadata, но бизнес-состояние, owner, статус scan/processing, размер, checksum, связи с сущностями - в БД.

**S3 strong consistent или eventually consistent?**

> Современный AWS S3 дает strong read-after-write consistency для `GET`, `PUT`, `LIST` и metadata/tag/ACL operations. Но это не дает транзакций между несколькими объектами и не устраняет задержки CDN или внешних индексов.

**Что спросить при выборе S3 vs MinIO?**

> SLA, compliance, data residency, стоимость egress, команда эксплуатации, backup/DR, нагрузка, latency, S3 feature compatibility, мониторинг, обновления, требования к on-prem и vendor lock-in.

## Архитектурный шаблон SaaS media storage

```text
Client
  -> Backend: request upload
  -> S3 private tmp: direct multipart upload
  -> Backend: complete upload
  -> Queue: scan/process job
  -> Worker: AV scan + metadata + thumbnails
  -> S3 private/public final prefixes
  -> DB: file status ready + active keys
  -> CDN: delivery with cache/signed access
```

Минимальная таблица `media_files`:

```text
id
owner_type
owner_id
disk
bucket
original_key
status
visibility
mime_type
size_bytes
checksum
metadata_json
created_by
created_at
updated_at
```

Статусы:

```text
pending_upload
uploaded
pending_scan
infected
processing
ready
failed
deleted
```

## Checklist

- Bucket private by default.
- Block Public Access включен.
- IAM policies минимальны.
- Разделены environments и credentials.
- Key schema не содержит PII.
- Есть upload state machine в БД.
- Есть validation реального MIME/размера/checksum.
- Есть virus scanning для user-generated files.
- Heavy processing вынесен в queue/workers.
- Для больших файлов используется direct/multipart upload.
- Настроен abort incomplete multipart uploads.
- Настроены lifecycle policies для tmp/old versions/archive.
- CDN origin закрыт от прямого public access.
- Для CDN настроены `Cache-Control` и signed access при необходимости.
- Mutable CDN keys не используются или имеют управляемый TTL/invalidation.
- Versioning/lifecycle согласованы с backup strategy.
- Есть мониторинг storage usage, request rate, errors, egress, KMS.
- Есть restore drill для критичных данных.
- Laravel/Symfony config не публикует private файлы случайно.
- Ошибки SDK логируются с request id.
- Jobs идемпотентны и retry-safe.

## Mini-practice

### Видео 2-5 GiB в Laravel API

- presigned multipart upload;
- upload session в БД;
- лимиты размера и типа;
- complete endpoint;
- `HEAD`/checksum verification;
- queue job для scan/transcode;
- lifecycle abort incomplete multipart;
- CDN delivery после статуса `ready`.

### CDN отдает старый аватар

Разобрать:

- invalidation;
- короткий TTL;
- immutable key;
- query string cache key;
- обновление URL в БД;
- влияние browser cache.

Лучший senior-ответ: перейти на immutable keys и долгий cache TTL.

### Медицинские PDF

- private bucket;
- Block Public Access;
- SSE-KMS;
- audit logs;
- strict IAM;
- short-lived signed access;
- virus scanning;
- retention policy;
- backup/DR;
- запрет PII в key;
- compliance requirements.

## Self-check

1. Чем object storage отличается от file/block storage?
2. Почему key с `/` не является настоящей директорией?
3. Какие данные стоит хранить в БД, а какие в object metadata?
4. Что означает strong consistency в S3 и чего она не гарантирует?
5. Почему presigned URL не заменяет авторизацию?
6. Когда нужен multipart upload?
7. Почему incomplete multipart uploads опасны?
8. Как защитить private bucket за CDN?
9. Когда использовать signed URL, а когда signed cookies?
10. Почему immutable keys лучше invalidation?
11. Как построить pipeline virus scan и thumbnails?
12. Почему нельзя доверять `Content-Type` от клиента?
13. Чем durability отличается от backup?
14. Какие cost drivers есть у S3/CDN?
15. Какие особенности MinIO важны в production?

## Ссылки

- [Amazon S3 Documentation](https://docs.aws.amazon.com/s3/)
- [Amazon S3 User Guide](https://docs.aws.amazon.com/AmazonS3/latest/userguide/Welcome.html)
- [Amazon S3 consistency model](https://docs.aws.amazon.com/AmazonS3/latest/userguide/Welcome.html#ConsistencyModel)
- [Amazon S3 presigned URLs](https://docs.aws.amazon.com/AmazonS3/latest/userguide/ShareObjectPreSignedURL.html)
- [Amazon S3 multipart upload](https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html)
- [Amazon S3 lifecycle configuration](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lifecycle-mgmt.html)
- [Amazon S3 bucket policies](https://docs.aws.amazon.com/AmazonS3/latest/userguide/bucket-policies.html)
- [Amazon S3 server-side encryption](https://docs.aws.amazon.com/AmazonS3/latest/userguide/serv-side-encryption.html)
- [Amazon S3 Object Lock](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lock.html)
- [Amazon CloudFront Documentation](https://docs.aws.amazon.com/cloudfront/)
- [CloudFront private content with signed URLs and signed cookies](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/PrivateContent.html)
- [CloudFront invalidation](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/Invalidation.html)
- [MinIO Documentation](https://min.io/docs/minio/linux/index.html)
- [MinIO S3 compatibility](https://min.io/docs/minio/linux/reference/s3-api-compatibility.html)
- [Laravel Filesystem](https://laravel.com/docs/filesystem)
- [Flysystem AWS S3 adapter](https://flysystem.thephpleague.com/docs/adapter/aws-s3-v3/)
- [Symfony Messenger](https://symfony.com/doc/current/messenger.html)
- [AWS SDK for PHP S3](https://docs.aws.amazon.com/sdk-for-php/v3/developer-guide/s3-examples.html)
