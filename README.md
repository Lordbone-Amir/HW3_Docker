# Счётчик в Docker

## Запуск через Compose

Выполните команды из корня проекта.

```sh
cp .env.example .env
docker compose up -d --build --wait
curl -fsS http://127.0.0.1:8080/
curl -fsS -X POST http://127.0.0.1:8080/increment
```

В `.env` можно изменить `GREETING` (приветствие) и `APP_PORT` (порт на компьютере). Внутри контейнера приложение слушает порт 8000.

Приложение подключается к Redis по адресу `redis://redis:6379/0`. Redis доступен внутри сети Compose и сохраняет журнал AOF в именованном томе `redis_data`, подключённом к `/data`. Compose ждёт ответа от Redis перед запуском приложения. Готовность приложения проверяется через `/readyz`.

```sh
docker compose ps
docker compose logs app
docker compose down
```

После `docker compose down` том остаётся на месте. Следующий запуск продолжит счёт с сохранённого значения.

[Отчёт о проверках](REPORT.md) · [Описание приложения](app/README.md) · [Условие задания](TASK.md)
