# Практика 4 и ДЗ 4. Журналы всех сервисов — в папке ./logs

Решение с практики 21.10: каждый сервис сам пишет журналы в папку проекта `./logs/<сервис>`. Папка переживает пересоздание контейнеров, журналы открываются обычным редактором, а на следующем шаге их соберёт Grafana. Методичка — по шагам, у каждого шага проверка. Ротацию и потолки по диску пока не делаем — это следующий шаг.

## Карта: кто куда пишет
| Сервис | По умолчанию | Что меняем | Путь в контейнере | Папка на хосте | Владелец (uid) |
|---|---|---|---|---|---|
| nginx | ссылки на stdout/stderr | подключаем папку поверх ссылок | `/var/log/nginx` | `logs/nginx` | root (главный процесс) |
| php-fpm | stderr (`/proc/self/fd/2`) | свой конфиг `zz-logs.conf` | `/var/log/php-fpm` | `logs/php-fpm` | 33 (`www-data`) |
| Laravel | `storage/logs/laravel.log` | канал `daily` + подключение папки | `/var/www/html/storage/logs` | `logs/laravel` | 33 (`www-data`) |
| FastAPI (uvicorn) | stdout/stderr | `--log-config` с файлами | `/var/log/api` | `logs/api` | root |
| MySQL | stderr | параметр запуска | `/var/log/mysql` | `logs/mysql` | 999 (`mysql`) |
| Redis | stdout | `--logfile` | `/var/log/redis` | `logs/redis` | 999:1000 (`redis`) |

Uid проверены в образах: `docker run --rm --entrypoint id <образ> <пользователь>`.

## Шаг 1. Папки и права — одним разовым контейнером
В `docker-compose.yml` добавьте сервис:
```yaml
  logs-init:
    image: alpine:3.20
    command: >
      sh -c "mkdir -p /logs/nginx /logs/php-fpm /logs/laravel /logs/api /logs/mysql /logs/redis &&
             chown -R 33:33 /logs/php-fpm /logs/laravel &&
             chown -R 999:999 /logs/mysql &&
             chown -R 999:1000 /logs/redis &&
             chmod 750 /logs/*"
    volumes:
      - ./logs:/logs
```
И каждому из шести сервисов — зависимость, чтобы он стартовал после того, как папки готовы:
```yaml
    depends_on:
      logs-init:
        condition: service_completed_successfully
```
В `.gitignore`: строка `logs/`.

`chown 33:33` — это «пользователь:группа» по номерам. Номера, а не имена: на хосте пользователя `www-data` может не быть.

*Проверка:* `rm -rf logs && docker compose up -d && ls -ln logs` — у папок владельцы 0, 33, 999 по таблице.

## Шаг 2. nginx
`docker-compose.yml`, сервис `nginx`, в `volumes`:
```yaml
      - ./logs/nginx:/var/log/nginx
```
В образе `/var/log/nginx/access.log` и `error.log` — ссылки на stdout и stderr. Подключённая папка их закрывает, и nginx пишет настоящие файлы. Дополнительно — JSON-формат с меткой запроса, в `infra/nginx/default.conf` над `server { … }`:
```nginx
log_format json escape=json '{"time":"$time_iso8601","remote_addr":"$remote_addr",'
  '"request_id":"$request_id","method":"$request_method","uri":"$request_uri",'
  '"status":$status,"bytes":$body_bytes_sent,"request_time":$request_time,'
  '"upstream_time":"$upstream_response_time","user_agent":"$http_user_agent"}';
```
и внутри `server { … }`:
```nginx
    access_log /var/log/nginx/access.log json;
    error_log  /var/log/nginx/error.log warn;
```
*Проверка:* открыть ленту → `tail -n 1 logs/nginx/access.log` — строка JSON с `request_id`.

## Шаг 3. Laravel
`docker-compose.yml`, сервис `php`:
```yaml
    environment:
      LOG_CHANNEL: daily
      LOG_DAILY_DAYS: "14"
    volumes:
      - ./logs/laravel:/var/www/html/storage/logs
```
Путь справа — **полный путь внутри контейнера**. Относительный `storage/logs` не работает: на практике журналы не появлялись именно из-за этого.

*Проверка:* `docker compose exec php php artisan tinker --execute="Log::warning('проверка журнала');"` → `ls logs/laravel/` — файл `laravel-ГГГГ-ММ-ДД.log` со строкой.

## Шаг 4. FastAPI (uvicorn)
Основной вариант — конфиг журналов Python. Файл `infra/api/logging.json`:
```json
{
  "version": 1,
  "disable_existing_loggers": false,
  "formatters": {"plain": {"format": "%(asctime)s %(levelname)s %(name)s %(message)s"}},
  "handlers": {
    "app_file": {"class": "logging.FileHandler", "filename": "/var/log/api/app.log",
                 "encoding": "utf-8", "formatter": "plain"},
    "access_file": {"class": "logging.FileHandler", "filename": "/var/log/api/access.log",
                    "encoding": "utf-8", "formatter": "plain"},
    "errors_stderr": {"class": "logging.StreamHandler", "stream": "ext://sys.stderr", "level": "ERROR", "formatter": "plain"}
  },
  "loggers": {
    "uvicorn":        {"handlers": ["app_file", "errors_stderr"], "level": "INFO", "propagate": false},
    "uvicorn.error":  {"handlers": ["app_file", "errors_stderr"], "level": "INFO", "propagate": false},
    "uvicorn.access": {"handlers": ["access_file"], "level": "INFO", "propagate": false}
  },
  "root": {"handlers": ["app_file", "errors_stderr"], "level": "INFO"}
}
```
`docker-compose.yml`, сервис `api`:
```yaml
    command: uvicorn app.main:app --host 0.0.0.0 --port 8000 --log-config /etc/api/logging.json
    volumes:
      - ./infra/api/logging.json:/etc/api/logging.json:ro
      - ./logs/api:/var/log/api
    environment:
      FORWARDED_ALLOW_IPS: "*"   # в журнал — адрес клиента, а не nginx
      PYTHONUNBUFFERED: "1"      # print без задержки
```
Вариант с практики — `sh -c "uvicorn … 2>&1 | tee -a /var/log/api/uvicorn.log"` — тоже засчитывается: журнал и в файле, и в консоли. Перенаправление `>` или `|` прямо в `command` без `sh -c` не работает — без оболочки эти знаки уходят в uvicorn как обычные аргументы.

*Проверка:* `curl -s -o /dev/null http://localhost:8080/api/health` → `tail -n 1 logs/api/access.log`.

## Шаг 5. php-fpm
Где он живёт: внутри контейнера `php`. Программа — `/usr/local/sbin/php-fpm`, его настройки — `/usr/local/etc/php-fpm.d/*.conf`, они читаются по алфавиту, последний побеждает. Поэтому свой файл называем `zz-…`.

Файл `infra/php/zz-logs.conf`:
```ini
[global]
error_log = /var/log/php-fpm/error.log

[www]
access.log = /var/log/php-fpm/access.log
catch_workers_output = yes
decorate_workers_output = no
```
`docker-compose.yml`, сервис `php`, в `volumes`:
```yaml
      - ./infra/php/zz-logs.conf:/usr/local/etc/php-fpm.d/zz-logs.conf:ro
      - ./logs/php-fpm:/var/log/php-fpm
```
*Проверка:*
```bash
docker compose up -d php
docker compose exec php php-fpm -tt 2>&1 | grep -E "error_log|access.log"
ls logs/php-fpm/
```

## Шаг 6. MySQL
`docker-compose.yml`, сервис `mysql`:
```yaml
    command:
      - --log-error=/var/log/mysql/error.log
    volumes:
      - mysql-data:/var/lib/mysql
      - ./logs/mysql:/var/log/mysql
```
Если у папки неправильный владелец, MySQL не сможет открыть журнал — поэтому шаг 1.

*Проверка:* `docker compose restart mysql` → `tail logs/mysql/error.log` — строки запуска.

## Шаг 7. Redis
```yaml
  redis:
    image: redis:7-alpine
    command: redis-server --logfile /var/log/redis/redis.log
    volumes:
      - ./logs/redis:/var/log/redis
```
*Проверка:* `docker compose restart redis` → `tail logs/redis/redis.log`.

## Где легко ошибиться
| Симптом | Причина | Что делать |
|---|---|---|
| Папка журнала пустая | неверный путь справа в `volumes` | полный путь внутри контейнера, сверить по таблице |
| `Permission denied` в журнале сервиса | у папки не тот владелец | шаг 1; `ls -ln logs` |
| uvicorn пишет только в консоль | `>` или `\|` в `command` без `sh -c` | вариант с `--log-config` или `sh -c "…"` |
| Настройки php-fpm не применились | файл не в `php-fpm.d/` или имя идёт раньше `docker.conf` | имя `zz-…`, проверка `php-fpm -tt` |
| Журнал MySQL не появляется | нет прав на папку | шаг 1; `ls -ln logs` |
| Журналы попали в git | нет `logs/` в `.gitignore` | добавить и `git rm -r --cached logs` |
