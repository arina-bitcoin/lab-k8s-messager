# Messager

Микросервисный мессенджер на Go + PostgreSQL.

## Быстрый старт (без исходников)

Нужен только Docker. Скачай три файла и запусти:

```bash
# 1. Скачай файлы
curl -O https://raw.githubusercontent.com/mablinov2704/own-messager/main/docker-compose.hub.yml
curl -O https://raw.githubusercontent.com/mablinov2704/own-messager/main/init-db.sh
mkdir -p user-service/migrations message-service/migrations
curl -o user-service/migrations/001_init.sql \
  https://raw.githubusercontent.com/mablinov2704/own-messager/main/user-service/migrations/001_init.sql
curl -o message-service/migrations/001_init.sql \
  https://raw.githubusercontent.com/mablinov2704/own-messager/main/message-service/migrations/001_init.sql
curl -o message-service/migrations/002_add_file_name.sql \
  https://raw.githubusercontent.com/mablinov2704/own-messager/main/message-service/migrations/002_add_file_name.sql

# 2. Запусти
docker compose -f docker-compose.hub.yml up -d
```

Открыть браузер: **http://localhost:8080**

## Docker Hub образы

| Образ | Платформы |
|-------|-----------|
| [`mablinov2704/frontend:latest`](https://hub.docker.com/r/mablinov2704/frontend) | linux/amd64, linux/arm64 |
| [`mablinov2704/bff:latest`](https://hub.docker.com/r/mablinov2704/bff) | linux/amd64, linux/arm64 |
| [`mablinov2704/user-service:latest`](https://hub.docker.com/r/mablinov2704/user-service) | linux/amd64, linux/arm64 |
| [`mablinov2704/message-service:latest`](https://hub.docker.com/r/mablinov2704/message-service) | linux/amd64, linux/arm64 |

### Переменные окружения frontend

| Переменная | По умолчанию | Описание |
|------------|-------------|----------|
| `BFF_URL` | `""` | Публичный URL BFF для браузера. Пусто = same-origin (nginx проксирует сам). Установите `http://your-host:8080` если frontend и BFF на разных хостах/портах. |
| `BFF_INTERNAL_URL` | `http://bff:8080` | URL BFF внутри Docker-сети для nginx proxy_pass. |

## Запуск из исходников

```bash
git clone <repo>
cd own-messager
docker compose up --build
```

## Архитектура

```
own-messager/
├── user-service/      # Регистрация и поиск пользователей (порт 8081)
├── message-service/   # Сообщения и файлы (порт 8082)
├── bff/               # BFF — агрегирует API, long polling, отдаёт фронтенд (порт 8080)
├── frontend/          # HTML/JS фронтенд
├── docker-compose.yml       # сборка из исходников
└── docker-compose.hub.yml   # готовые образы с Docker Hub
```

## API (через BFF, порт 8080)

| Метод | Путь | Описание |
|-------|------|----------|
| POST | `/api/v1/users` | Регистрация `{"name":"..."}` |
| GET | `/api/v1/users?q=...` | Поиск пользователей |
| GET | `/api/v1/users/:id` | Получить пользователя |
| POST | `/api/v1/messages` | Отправить сообщение |
| PUT | `/api/v1/messages/:id` | Изменить сообщение (только автор) |
| DELETE | `/api/v1/messages/:id?user_id=...` | Удалить сообщение (только автор) |
| GET | `/api/v1/messages?user_a=&user_b=` | История переписки |
| GET | `/api/v1/conversations?user_id=` | Список диалогов с последним сообщением |
| GET | `/api/v1/poll?user_a=&user_b=&after_id=` | Long polling новых сообщений |
| POST | `/api/v1/files` | Загрузить файл (multipart `file`) |
| GET | `/api/v1/files/:id` | Скачать файл |

## Разработка

```bash
cd user-service && make run
cd message-service && make run
cd bff && make run
```

Переменные окружения — `.env.example` в каждом сервисе.

## Миграции

Применяются автоматически при `docker compose up`.

Ручной запуск:
```bash
cd user-service && make migrate DB_DSN="postgres://messager:messager@localhost:5432/messager_users?sslmode=disable"
cd message-service && make migrate DB_DSN="postgres://messager:messager@localhost:5432/messager_messages?sslmode=disable"
```
