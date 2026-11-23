# Отчёт о проверках

## 4 балла. Один контейнер

```sh
docker build -t ipr-counter:local .
docker run -d --name ipr-counter-check -p 18080:8000 \
  -e 'GREETING=Привет из docker run!' ipr-counter:local
url=http://127.0.0.1:18080
curl -fsS --retry 5 --retry-all-errors --retry-delay 1 --max-time 4 "$url/healthz" >/dev/null
curl -fsS "$url/"
curl -fsS -X POST "$url/increment"
curl -fsS -X POST "$url/increment"
docker logs ipr-counter-check
docker rm -f ipr-counter-check
```

Ответы на запросы и вывод `docker logs` (время в UTC).

```text
{"greeting": "Привет из docker run!", "count": 0, "hostname": "eeb1c1a837b8", "storage": "sqlite"}
{"greeting": "Привет из docker run!", "count": 1, "hostname": "eeb1c1a837b8", "storage": "sqlite"}
{"greeting": "Привет из docker run!", "count": 2, "hostname": "eeb1c1a837b8", "storage": "sqlite"}
2026-11-23 21:44:59,037 listening port=8000 storage=sqlite
2026-11-23 21:44:59,921 http 172.17.0.1 "GET /healthz HTTP/1.1" 200 -
2026-11-23 21:44:59,935 http 172.17.0.1 "GET / HTTP/1.1" 200 -
2026-11-23 21:44:59,951 http 172.17.0.1 "POST /increment HTTP/1.1" 200 -
2026-11-23 21:44:59,965 http 172.17.0.1 "POST /increment HTTP/1.1" 200 -
```

## 6 баллов. Compose и SQLite

Для этого этапа использовал временный файл `compose-sqlite.yaml` со следующим содержимым. Поместите его рядом с `Dockerfile`, если хотите повторить проверку.

```yaml
services:
  app:
    build: .
    image: ipr-counter:local
    environment:
      GREETING: ${GREETING:-Hello, Docker!}
    ports:
      - "${APP_PORT:-8080}:8000"
    volumes:
      - sqlite_data:/data

volumes:
  sqlite_data:
```

```sh
export APP_PORT=18081
dc() {
  docker compose --env-file .env.example -f compose-sqlite.yaml -p ipr-sqlite-check "$@"
}
url=http://127.0.0.1:18081

dc up -d
curl -fsS --retry 5 --retry-all-errors "$url/healthz"
curl -fsS "$url/"
curl -fsS -X POST "$url/increment"
curl -fsS -X POST "$url/increment"
dc down
dc up -d
curl -fsS --retry 5 --retry-all-errors "$url/healthz"
curl -fsS "$url/"
```

Ответы счётчика. После пересоздания изменился `hostname`, а счётчик остался равен 2.

```text
{"greeting": "Привет, Docker!", "count": 0, "hostname": "5bee93c8c6b9", "storage": "sqlite"}
{"greeting": "Привет, Docker!", "count": 1, "hostname": "5bee93c8c6b9", "storage": "sqlite"}
{"greeting": "Привет, Docker!", "count": 2, "hostname": "5bee93c8c6b9", "storage": "sqlite"}
{"greeting": "Привет, Docker!", "count": 2, "hostname": "923c9d09482f", "storage": "sqlite"}
```

## 8 баллов. Redis

Проверял итоговый `compose.yaml` в отдельном Compose-проекте с пустым томом.

```sh
export APP_PORT=18082
dc() {
  docker compose --env-file .env.example -p ipr-redis-check "$@"
}
```

Вывод команд. Служебные строки сборки, создания и остановки контейнеров опущены.

```text
$ dc up -d --build --wait --wait-timeout 40
$ dc ps
NAME                      IMAGE                 COMMAND                  SERVICE   CREATED         STATUS                   PORTS
ipr-redis-check-app-1     ipr-redis-check-app   "python app.py"          app       5 seconds ago   Up 2 seconds (healthy)   0.0.0.0:18082->8000/tcp, [::]:18082->8000/tcp
ipr-redis-check-redis-1   redis:7.4-alpine      "docker-entrypoint.s…"   redis     5 seconds ago   Up 5 seconds (healthy)   6379/tcp
$ curl -sS --max-time 5 -X GET -w "\nHTTP %{http_code}\n" http://127.0.0.1:18082/
{"greeting": "Привет, Docker!", "count": 0, "hostname": "f6afa36a5273", "storage": "redis"}
HTTP 200
$ curl -sS --max-time 5 -X POST -w "\nHTTP %{http_code}\n" http://127.0.0.1:18082/increment
{"greeting": "Привет, Docker!", "count": 1, "hostname": "f6afa36a5273", "storage": "redis"}
HTTP 200
$ curl -sS --max-time 5 -X POST -w "\nHTTP %{http_code}\n" http://127.0.0.1:18082/increment
{"greeting": "Привет, Docker!", "count": 2, "hostname": "f6afa36a5273", "storage": "redis"}
HTTP 200
$ curl -sS --max-time 5 -X POST -w "\nHTTP %{http_code}\n" http://127.0.0.1:18082/increment
{"greeting": "Привет, Docker!", "count": 3, "hostname": "f6afa36a5273", "storage": "redis"}
HTTP 200
$ curl -sS --max-time 5 -X GET -w "\nHTTP %{http_code}\n" http://127.0.0.1:18082/
{"greeting": "Привет, Docker!", "count": 3, "hostname": "f6afa36a5273", "storage": "redis"}
HTTP 200
$ dc down
$ dc up -d --wait --wait-timeout 40
$ curl -sS --max-time 5 -X GET -w "\nHTTP %{http_code}\n" http://127.0.0.1:18082/
{"greeting": "Привет, Docker!", "count": 3, "hostname": "8b6960d3b72a", "storage": "redis"}
HTTP 200
$ docker inspect --format '{{.Id}} {{.State.StartedAt}}' "$(dc ps -q app)"
8b6960d3b72ade03e11e5440b29a72981857632083da1eed0bc5a31ae53043f4 2026-11-23T21:47:29.458126653Z
$ dc stop redis
$ curl -sS --max-time 5 -X GET -w "\nHTTP %{http_code}\n" http://127.0.0.1:18082/healthz
{"status": "alive"}
HTTP 200
$ curl -sS --max-time 5 -X GET -w "\nHTTP %{http_code}\n" http://127.0.0.1:18082/readyz
{"error": "storage_unavailable"}
HTTP 503
$ curl -sS --max-time 5 -X POST -w "\nHTTP %{http_code}\n" http://127.0.0.1:18082/increment
{"error": "storage_unavailable"}
HTTP 503
$ dc start redis
$ curl -fsS --retry 10 --retry-all-errors --retry-delay 1 --max-time 5 http://127.0.0.1:18082/readyz
{"status": "ready", "storage": "redis"}
$ curl -sS --max-time 5 -X GET -w "\nHTTP %{http_code}\n" http://127.0.0.1:18082/
{"greeting": "Привет, Docker!", "count": 3, "hostname": "8b6960d3b72a", "storage": "redis"}
HTTP 200
$ curl -sS --max-time 5 -X POST -w "\nHTTP %{http_code}\n" http://127.0.0.1:18082/increment
{"greeting": "Привет, Docker!", "count": 4, "hostname": "8b6960d3b72a", "storage": "redis"}
HTTP 200
$ docker inspect --format '{{.Id}} {{.State.StartedAt}}' "$(dc ps -q app)"
8b6960d3b72ade03e11e5440b29a72981857632083da1eed0bc5a31ae53043f4 2026-11-23T21:47:29.458126653Z
```

После `down` и `up` счётчик остался равен 3. Пока Redis был остановлен, `/healthz` отвечал 200, а `/readyz` и `POST /increment` возвращали 503. После запуска Redis счётчик снова показал 3, следующий POST вернул 4. Идентификатор и время запуска контейнера приложения до и после остановки Redis совпали.
