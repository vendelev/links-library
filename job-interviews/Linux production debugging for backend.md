# Linux production debugging для backend

Целевая аудитория: Lead/Senior PHP backend developer, который отвечает не только за код, но и за продакшен-инциденты: PHP-FPM, очереди, cron, systemd, Docker, сеть, диски, лимиты ОС и базовую observability.

Главная идея на собеседовании: не угадывать причину, а быстро сузить область проблемы по симптомам, метрикам и фактам.

## Ментальная модель инцидента

1. Что именно сломалось: endpoint, воркеры, cron, весь хост, контейнер, сеть, диск, БД, внешняя интеграция.
2. Когда началось: релиз, деплой, миграция, cron job, рост трафика, ротация логов, истечение сертификата, изменение DNS.
3. Насколько широко: один pod/container/host, один AZ/DC, одна версия приложения, один tenant, все пользователи.
4. Что ограничено: CPU, память, диск, inode, сеть, file descriptors, PHP-FPM workers, DB connections, external API latency.
5. Какой безопасный mitigation: rollback, scale out, restart конкретного сервиса, отключение тяжелой джобы, rate limit, feature flag, очистка диска, увеличение лимитов.

## Быстрый triage

```bash
date
hostname
uptime
who
top
free -h
df -h
df -ih
ss -s
systemctl --failed
journalctl -p warning..alert --since "30 min ago"
```

Что смотреть первым:

- `uptime`: load average и длительность проблемы.
- `top`: CPU, memory, zombie, steal time, конкретные процессы.
- `free -h`: доступная память, swap.
- `df -h` и `df -ih`: место и inode.
- `ss -s`: количество TCP-соединений и состояния.
- `journalctl`: OOM, segmentation fault, permission denied, restart loops.

Senior-ответ: «Я начинаю не с перезапуска, а с определения bottleneck: CPU, память, диск, сеть, лимиты или приложение. Параллельно фиксирую timeline и ищу безопасный mitigation».

## Процессы

### `ps`

```bash
ps aux --sort=-%cpu | head
ps aux --sort=-%mem | head
ps -eo pid,ppid,user,stat,pcpu,pmem,etime,cmd --sort=-pcpu | head
ps -ef | grep php-fpm
```

Важные поля:

- `PID`: процесс.
- `PPID`: родитель.
- `STAT`: состояние.
- `R`: running.
- `S`: sleeping.
- `D`: uninterruptible sleep, часто I/O wait.
- `Z`: zombie.
- `TIME`: накопленное CPU-время.
- `ETIME`: сколько процесс живет.

Типовые выводы:

- Много `php-fpm` с долгим `ETIME` и высоким CPU: тяжелые PHP-запросы, бесконечный цикл, неоптимальный код.
- Много процессов в `D`: проблема диска, NFS, storage, медленный I/O.
- Zombie сами по себе обычно симптом, а не причина; важнее родительский процесс.

### `top` и `htop`

```bash
top
top -H -p <pid>
htop
```

Что смотреть:

- `%us`: user CPU, приложение.
- `%sy`: kernel CPU, syscalls, сеть, filesystem.
- `%wa`: I/O wait.
- `%st`: steal time на виртуализации.
- `RES`: resident memory.
- `VIRT`: виртуальная память, не равна реальному потреблению.
- `Load average`: очередь runnable или uninterruptible задач, не просто CPU.

Pitfall: load average 20 на машине с 32 CPU может быть нормальным, а load 8 на машине с 2 CPU уже серьезный сигнал.

## CPU и load average

```bash
uptime
mpstat 1
vmstat 1
pidstat 1
```

Если `mpstat` и `pidstat` не установлены, часто пакет называется `sysstat`.

`vmstat 1`:

```text
r  b  swpd  free  buff  cache  si  so  bi  bo  in  cs  us sy id wa st
```

Что важно:

- `r`: runnable queue. Если стабильно выше числа CPU, есть CPU contention.
- `b`: blocked tasks. Если растет, вероятен I/O bottleneck.
- `si/so`: swap in/out. Активный swap почти всегда плохо для latency.
- `us/sy/wa/st`: user, system, iowait, steal.

Диагностика высокого load:

1. Сравнить load с числом CPU: `nproc`.
2. Посмотреть `top`: CPU busy или `wa`/`st`.
3. Найти процессы: `ps`, `top`, `pidstat`.
4. Проверить disk I/O: `iostat -xz 1`.
5. Проверить PHP-FPM slowlog и access log.
6. Проверить недавние cron jobs, деплой, очереди.

Senior-ответ: «High load не означает автоматически high CPU. Я проверяю runnable queue, blocked tasks, iowait и steal time. Если процессы в `D`, перезапуск приложения может не помочь, нужно смотреть storage».

## Память

### `free`

```bash
free -h
free -m
```

Смотреть:

- `available`: сколько памяти реально можно использовать без swap.
- `buff/cache`: page cache, обычно не проблема.
- `swap used`: важно не само наличие swap, а активный `si/so` в `vmstat`.

Pitfall: строка `free` не означает «память закончилась», Linux активно использует память под cache.

### Поиск потребителей памяти

```bash
ps aux --sort=-%mem | head -20
pmap -x <pid> | tail
cat /proc/<pid>/status
```

Полезные поля `/proc/<pid>/status`:

- `VmRSS`: resident memory.
- `VmSize`: virtual memory.
- `Threads`: число потоков.
- `FDSize`: размер таблицы file descriptors.

### OOM killer

```bash
journalctl -k --since "2 hours ago" | grep -i oom
dmesg -T | grep -i -E 'oom|killed process'
```

Признаки:

- В логах есть `Out of memory`.
- `Killed process <pid> (php-fpm)`.
- Сервис внезапно рестартовал без PHP exception.

Что объяснить на интервью:

- OOM killer выбирает процесс по `oom_score`, потреблению памяти и политике cgroups.
- В Docker/Kubernetes процесс может быть убит лимитом контейнера, даже если на хосте есть память.
- Нужно смотреть не только системные логи, но и container runtime / orchestrator events.

Mitigation:

- Откатить релиз с memory leak.
- Уменьшить concurrency.
- Ограничить тяжелые jobs.
- Увеличить memory limit контейнера/хоста.
- Проверить `memory_limit` PHP и размер responses/batches.

## Диск

### Свободное место и inode

```bash
df -h
df -ih
du -sh /var/log/* 2>/dev/null
du -xh --max-depth=1 /var | sort -h
```

`df -h` показывает занятое место на filesystem. `du` показывает размер файлов по дереву. Они могут расходиться.

Почему `df` и `du` отличаются:

- Удаленный файл еще открыт процессом.
- Mount namespace контейнера.
- Файлы скрыты под mount point.
- Sparse files.

Найти удаленные, но открытые файлы:

```bash
lsof +L1
lsof | grep deleted
```

Mitigation при disk full:

1. Не удалять вслепую важные данные.
2. Найти крупные директории: logs, cache, uploads, tmp, Docker overlay.
3. Проверить inode: много мелких файлов может заполнить inode при свободном месте.
4. Если удаленный лог удерживается процессом, перезапустить или переоткрыть лог у процесса.
5. Настроить logrotate/retention.

### I/O диагностика

```bash
iostat -xz 1
vmstat 1
```

`iostat -xz`:

- `%util`: загрузка устройства, не всегда равно saturation для SSD/NVMe.
- `await`: средняя latency I/O.
- `r/s`, `w/s`: операции чтения/записи.
- `rkB/s`, `wkB/s`: throughput.
- `aqu-sz`: размер очереди.

Типовые причины высокого I/O:

- Логирование слишком подробное.
- Большие batch jobs.
- Backup, compression, rotation.
- Docker overlay2 разросся.
- База данных на том же диске.
- NFS/S3 fuse/сетевой storage.

## File descriptors, `lsof`, `ulimit`

```bash
ulimit -n
cat /proc/<pid>/limits
ls /proc/<pid>/fd | wc -l
lsof -p <pid>
lsof -i
```

Симптомы исчерпания FD:

- `Too many open files`.
- PHP-FPM не принимает новые соединения.
- Nginx не может открыть socket/log/cache file.
- Много соединений в `ESTABLISHED`, `CLOSE_WAIT`.

Где лимиты задаются:

- shell: `ulimit`.
- systemd unit: `LimitNOFILE=`.
- Docker: `--ulimit nofile=...`.
- PHP-FPM pool и master process наследуют лимиты от service manager.

Senior-ответ: «Если приложение падает с `Too many open files`, я смотрю текущие лимиты процесса через `/proc/<pid>/limits`, фактическое число fd через `/proc/<pid>/fd`, типы fd через `lsof`, а не только `ulimit` в своей shell-сессии».

## Сеть

### `ss` и `netstat`

```bash
ss -tulpn
ss -tan state established
ss -tan state time-wait
ss -tan state close-wait
ss -s
netstat -tulpn
```

`ss` обычно предпочтительнее `netstat`: быстрее и современнее. `netstat` полезен на старых системах.

Состояния TCP:

- `LISTEN`: процесс слушает порт.
- `ESTABLISHED`: активное соединение.
- `TIME_WAIT`: нормальное состояние после закрытия, но массово может указывать на connection churn.
- `CLOSE_WAIT`: remote закрыл соединение, local app не закрыла socket. Часто bug/leak в приложении.
- `SYN_SENT`: не удается установить исходящее соединение.
- `SYN_RECV`: входящие handshake, возможна перегрузка или SYN flood.

### `curl`

```bash
curl -v https://example.com/
curl -I https://example.com/
curl -sS -o /dev/null -w 'dns=%{time_namelookup} connect=%{time_connect} tls=%{time_appconnect} ttfb=%{time_starttransfer} total=%{time_total}\n' https://example.com/
curl --resolve example.com:443:1.2.3.4 https://example.com/
```

Что диагностировать:

- DNS latency.
- TCP connect latency.
- TLS handshake.
- TTFB.
- HTTP status.
- Header differences.
- Проблемы через proxy/load balancer.

### `dig`

```bash
dig example.com
dig +short example.com
dig @8.8.8.8 example.com
dig example.com A
dig example.com AAAA
dig example.com MX
```

Проверять:

- Корректность DNS-записей.
- TTL.
- Расхождение между resolvers.
- IPv4/IPv6 различия.
- Split-horizon DNS в корпоративной сети.

### `tcpdump`

```bash
tcpdump -i any port 443
tcpdump -i eth0 host 1.2.3.4
tcpdump -i any -nn 'tcp port 5432'
tcpdump -i any -w capture.pcap 'host 1.2.3.4 and port 443'
```

Базовые правила:

- Использовать `-nn`, чтобы не тратить время на reverse DNS/service names.
- Фильтровать по host/port, иначе легко собрать слишком много данных.
- Не сохранять чувствительный трафик без необходимости.
- Для HTTPS payload не виден, но handshake, retransmits и resets видны.

Признаки:

- SYN без SYN-ACK: сеть/firewall/remote недоступен.
- Retransmissions: packet loss или перегрузка.
- RST: соединение активно сброшено одной из сторон.

## Логи

Типовые места:

```bash
/var/log/syslog
/var/log/messages
/var/log/auth.log
/var/log/nginx/access.log
/var/log/nginx/error.log
/var/log/php*-fpm.log
/var/log/cron
/var/log/docker.log
```

Команды:

```bash
journalctl -u nginx --since "1 hour ago"
journalctl -u php-fpm --since "today"
journalctl -f
journalctl -k
journalctl -p err..alert
```

Что важно для Senior:

- Коррелировать request id/trace id между Nginx, PHP, приложением, очередями и БД.
- Сравнивать время в UTC/local timezone.
- Не полагаться только на application logs: kernel/systemd/container logs часто дают причину рестарта.
- Проверять log rotation: потеря логов, забитый диск, процесс пишет в deleted file.

## systemd

```bash
systemctl status php-fpm
systemctl status nginx
systemctl list-units --failed
systemctl cat php-fpm
systemctl show php-fpm --property=MainPID,LimitNOFILE,Restart,MemoryMax,CPUQuota
journalctl -u php-fpm --since "2 hours ago"
```

Что смотреть:

- `Active`: active, failed, activating.
- `MainPID`: реальный PID процесса.
- `Restart`: политика рестарта.
- `ExecStart`: фактическая команда запуска.
- `LimitNOFILE`, `MemoryMax`, `CPUQuota`: лимиты.
- Restart loop: сервис постоянно падает и перезапускается.

Pitfall: `systemctl restart` может временно скрыть проблему. Перед рестартом лучше собрать минимум фактов: `status`, последние логи, PID, лимиты, core/OOM признаки.

## cron

```bash
crontab -l
sudo crontab -u www-data -l
ls -la /etc/cron.*
systemctl list-timers
journalctl -u cron --since "today"
```

Проблемы cron:

- Job запускается одновременно с предыдущей копией.
- Отличается environment: нет нужного `PATH`, `.env`, locale, working directory.
- Output не перенаправлен и теряется или отправляется mail.
- Нет locking.
- Job стартует во время пикового трафика.
- В контейнере cron не запущен или запущен не там.

Senior-ответ: «Для периодических задач я проверяю overlap, lock, duration, exit code, environment и нагрузку на shared resources. На собеседовании важно сказать, что cron-инциденты часто выглядят как внезапный рост CPU/I/O в одно и то же время».

## Permissions

```bash
id
namei -l /path/to/file
ls -la /path/to/file
stat /path/to/file
getfacl /path/to/file
```

Проверять:

- Пользователь процесса: `www-data`, `nginx`, `deploy`, container user.
- Права на все директории по пути, не только на файл.
- Ownership после deploy.
- SELinux/AppArmor, если включены.
- Read/write/execute semantics для директорий.
- Sticky bit на `/tmp`.

Pitfall: `chmod 777` как «решение» на проде неприемлем. Senior должен найти минимально нужные права и владельца.

## `strace`

```bash
strace -p <pid>
strace -f -p <pid>
strace -tt -T -f -p <pid>
strace -e trace=file -p <pid>
strace -e trace=network -p <pid>
```

Что дает:

- Какие syscalls выполняет процесс.
- На каком файле/socket он блокируется.
- Permission denied, file not found.
- Медленные `connect`, `read`, `write`, `futex`, `openat`.

Осторожность:

- `strace` добавляет overhead.
- На горячем production-процессе использовать кратко и точечно.
- Лучше attach к одному worker, а не ко всему пулу.

Пример интерпретации:

- Много `connect()` к Redis с ошибкой: проблема connectivity или pool.
- `openat(... ENOENT)`: приложение ищет не тот файл/config.
- Долгий `read()` из socket: upstream медленно отвечает.
- Много `futex`: lock contention или ожидание потоков.

## PHP-FPM worker exhaustion

Симптомы:

- Nginx возвращает `502`, `503`, `504`.
- В логах: `server reached pm.max_children setting`.
- Все PHP-FPM workers заняты.
- Очередь запросов растет.
- Latency резко растет, CPU может быть не высоким.

Команды:

```bash
ps -o pid,ppid,user,stat,pcpu,pmem,etime,cmd -C php-fpm
ss -ltnp | grep php
systemctl status php-fpm
journalctl -u php-fpm --since "30 min ago"
```

Что проверить в конфиге pool:

```text
pm = dynamic|static|ondemand
pm.max_children
pm.start_servers
pm.min_spare_servers
pm.max_spare_servers
request_terminate_timeout
request_slowlog_timeout
slowlog
```

Диагностика:

1. Проверить, достигнут ли `pm.max_children`.
2. Посмотреть slowlog PHP-FPM.
3. Сопоставить с Nginx access log: какие endpoints медленные.
4. Проверить внешние зависимости: DB, Redis, HTTP API.
5. Проверить memory per worker: `pm.max_children * RSS` должен помещаться в память.
6. Проверить `listen.backlog` и очередь соединений.

Senior-ответ: «Я не просто увеличиваю `pm.max_children`. Сначала считаю память на worker и ищу, почему workers заняты: CPU-bound PHP, ожидание БД, HTTP API, Redis, filesystem или deadlock. Иначе можно получить OOM вместо 502».

## Docker

```bash
docker ps
docker logs --tail=200 <container>
docker logs -f <container>
docker exec -it <container> sh
docker stats
docker inspect <container>
```

Что смотреть:

- Restart count.
- Exit code.
- OOMKilled.
- Memory/CPU limits.
- Mounts/volumes.
- Network mode.
- Healthcheck status.
- Container time и timezone.

Полезные команды:

```bash
docker inspect <container> --format '{{.State.OOMKilled}} {{.State.ExitCode}} {{.State.RestartCount}}'
docker inspect <container> --format '{{json .HostConfig.Memory}}'
docker exec <container> ps aux
docker exec <container> df -h
docker exec <container> ss -tulpn
```

Pitfalls:

- Внутри контейнера может не быть `top`, `curl`, `dig`, `strace`.
- `localhost` внутри контейнера — сам контейнер, не хост.
- Memory limit контейнера отличается от памяти хоста.
- Логи контейнера могут забить диск хоста, если нет rotation.
- PID namespace может скрывать процессы хоста.

## Частые инциденты

### Disk full

Checklist:

1. `df -h`, `df -ih`.
2. `du -xh --max-depth=1 /var | sort -h`.
3. Проверить `/var/log`, `/tmp`, uploads, cache, Docker.
4. `lsof +L1` для deleted files.
5. Проверить logrotate и retention.
6. Безопасно освободить место или расширить volume.
7. Убедиться, что сервисы снова пишут логи и временные файлы.

Senior-пояснение: «Удаление файла не всегда освобождает место, если процесс держит fd. Тогда нужен restart/reopen конкретного процесса».

### High load

Checklist:

1. `uptime`, `nproc`.
2. `top`: CPU vs iowait vs steal.
3. `vmstat 1`: `r`, `b`, `si/so`, `wa`.
4. `ps aux --sort=-%cpu`, `ps aux --sort=-%mem`.
5. `iostat -xz 1`.
6. Проверить cron, очереди, релиз, batch jobs.
7. Проверить PHP-FPM slowlog и endpoints.

Senior-пояснение: «Load может расти от I/O wait и blocked tasks. Поэтому я смотрю состояния процессов и iowait, а не только CPU percent».

### Slow network / external API timeout

Checklist:

1. `curl -w` разбить latency на DNS/connect/TLS/TTFB.
2. `dig` проверить DNS и разные resolvers.
3. `ss -tan` посмотреть состояния соединений.
4. `tcpdump` проверить handshake/retransmits/RST.
5. Проверить proxy, firewall, security groups, route, IPv6.
6. Сравнить из контейнера и с хоста.
7. Проверить retries/timeouts/circuit breaker приложения.

Senior-пояснение: «Я разделяю проблему на DNS, TCP connect, TLS, server processing и transfer. `curl -w` часто быстрее всего показывает слой проблемы».

### PHP-FPM saturation

Checklist:

1. Проверить Nginx errors: `connect() to unix socket failed`, `upstream timed out`, `recv() failed`.
2. Проверить PHP-FPM logs: `pm.max_children`.
3. Посчитать активные workers.
4. Включить/посмотреть slowlog.
5. Проверить DB/Redis/external API latency.
6. Проверить memory per worker и риск OOM.
7. Mitigation: scale out, временно поднять limits, отключить тяжелый endpoint/job, rollback.

Senior-пояснение: «Насыщение PHP-FPM часто является следствием медленной зависимости. Увеличение workers может усилить pressure на DB и ухудшить ситуацию».

### OOM

Checklist:

1. `journalctl -k | grep -i oom`.
2. Проверить container state: `OOMKilled`.
3. `free -h`, `vmstat 1`.
4. `ps aux --sort=-%mem`.
5. Проверить релиз, batch size, memory leak, большие exports/imports.
6. Проверить PHP `memory_limit`, FPM `pm.max_children`, container memory limit.
7. Mitigation: уменьшить concurrency, rollback, поднять лимит, разбить batch.

Senior-пояснение: «OOM на хосте и OOM внутри cgroup отличаются. В контейнерах я всегда смотрю лимит и events/runtime, а не только `free` на хосте».

## Типовые ловушки на собеседовании

- «Перезапущу сервис» без сбора фактов: можно потерять evidence.
- «High load значит CPU»: нет, может быть I/O, lock, steal, uninterruptible sleep.
- «Память закончилась, потому что free мало»: Linux page cache не равен утечке.
- «Удалил лог, место освободится»: не освободится, если файл открыт процессом.
- «Увеличу `pm.max_children`»: можно получить OOM или перегрузить БД.
- «`ulimit -n` в shell показывает лимит сервиса»: нет, нужен `/proc/<pid>/limits` или `systemctl show`.
- «Проблема сети, потому что curl медленный»: нужно разделить DNS/connect/TLS/TTFB.
- «В контейнере localhost — хост»: нет, это сам контейнер.
- «`tcpdump` покажет HTTP body для HTTPS»: без TLS termination не покажет payload.

## Senior answers

### Как диагностировать 502 от Nginx к PHP-FPM?

Я смотрю Nginx error log, статус PHP-FPM service, socket/listen, наличие `pm.max_children`, slowlog и saturation workers. Затем проверяю, почему workers заняты: CPU, БД, Redis, external API, filesystem. Если нужен быстрый mitigation, масштабирую или временно поднимаю workers только после оценки памяти.

### Что делать при `Too many open files`?

Смотрю лимиты процесса через `/proc/<pid>/limits`, фактические fd через `/proc/<pid>/fd`, типы fd через `lsof -p`. Проверяю TCP states, особенно `CLOSE_WAIT`, и логи приложения. Исправление может быть в закрытии ресурсов, connection pooling, `LimitNOFILE` в systemd или Docker `--ulimit`, но лимит нельзя повышать без понимания утечки.

### Как отличить нехватку CPU от I/O bottleneck?

Смотрю `top` и `vmstat`: если высокий `us/sy` и большая runnable queue `r`, вероятен CPU. Если высокий `wa`, процессы в `D`, растет `b`, а `iostat` показывает высокий `await` и очередь, это I/O. Load average сам по себе недостаточен.

### Как действовать при OOMKilled контейнера?

Проверяю `docker inspect`/events, memory limit контейнера, логи приложения перед смертью, memory usage через `docker stats`, PHP-FPM `pm.max_children * RSS`, batch jobs и недавний релиз. Mitigation: уменьшить concurrency, поднять лимит, rollback, разбить обработку данных, включить streaming.

### Как проверить проблему с DNS?

Использую `dig` против системного resolver и внешнего resolver, проверяю A/AAAA, TTL, split-horizon, `/etc/resolv.conf`, различия между хостом и контейнером. Через `curl -w` смотрю `time_namelookup`.

## Мини-практика

1. На тестовом сервере найти топ-5 процессов по CPU и памяти через `ps`.
2. Сравнить `df -h` и `du -sh` для `/var/log`.
3. Запустить `curl -w` к внешнему HTTPS endpoint и разобрать DNS/connect/TLS/TTFB.
4. Посмотреть лимиты текущей shell и любого systemd-сервиса.
5. Найти слушающие порты через `ss -tulpn`.
6. Посмотреть последние kernel warnings через `journalctl -k`.
7. В Docker-контейнере сравнить `localhost` и имя соседнего сервиса в bridge network.
8. Включить PHP-FPM slowlog на staging и проверить, как выглядит запись медленного запроса.

## Self-check

- Чем `RES` отличается от `VIRT` в `top`?
- Почему load average может быть высоким при низком CPU?
- Что означает много процессов в состоянии `D`?
- Как найти удаленный файл, который все еще занимает место на диске?
- Почему `free` memory в Linux не равна реально доступной памяти?
- Где смотреть OOM killer events?
- Почему `CLOSE_WAIT` опаснее, чем большое количество `TIME_WAIT`?
- Как проверить, достиг ли PHP-FPM `pm.max_children`?
- Почему нельзя бездумно увеличивать число PHP-FPM workers?
- Чем лимит file descriptors в shell отличается от лимита systemd-сервиса?
- Как разделить сетевую задержку на DNS, connect, TLS и TTFB?
- Что проверить, если cron job работает вручную, но не работает из cron?
- Почему после удаления лог-файла место может не освободиться?
- Что проверить в Docker при внезапном рестарте контейнера?
- Когда использовать `strace`, а когда лучше не трогать production-процесс?

## Ссылки

- `man top`: https://man7.org/linux/man-pages/man1/top.1.html
- `man ps`: https://man7.org/linux/man-pages/man1/ps.1.html
- `man free`: https://man7.org/linux/man-pages/man1/free.1.html
- `man vmstat`: https://man7.org/linux/man-pages/man8/vmstat.8.html
- `man iostat`: https://man7.org/linux/man-pages/man1/iostat.1.html
- `man df`: https://man7.org/linux/man-pages/man1/df.1.html
- `man du`: https://man7.org/linux/man-pages/man1/du.1.html
- `man lsof`: https://man7.org/linux/man-pages/man8/lsof.8.html
- `man ss`: https://man7.org/linux/man-pages/man8/ss.8.html
- `man journalctl`: https://man7.org/linux/man-pages/man1/journalctl.1.html
- `man systemctl`: https://man7.org/linux/man-pages/man1/systemctl.1.html
- `man strace`: https://man7.org/linux/man-pages/man1/strace.1.html
- `man tcpdump`: https://www.tcpdump.org/manpages/tcpdump.1.html
- Brendan Gregg, Linux Performance: https://www.brendangregg.com/linuxperf.html
- Brendan Gregg, USE Method: https://www.brendangregg.com/usemethod.html
- Linux kernel OOM killer docs: https://docs.kernel.org/mm/oom.html
- Linux proc filesystem docs: https://docs.kernel.org/filesystems/proc.html
- systemd documentation: https://www.freedesktop.org/software/systemd/man/latest/
- Docker logs docs: https://docs.docker.com/reference/cli/docker/container/logs/
- Docker exec docs: https://docs.docker.com/reference/cli/docker/container/exec/
- Docker stats docs: https://docs.docker.com/reference/cli/docker/container/stats/
