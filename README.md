# Лабораторная работа: Запуск микросервисного приложения в Kubernetes

## Цель

Развернуть текущий проект мессенджера в Kubernetes-кластере, настроить хранение файлов через S3 CSI, организовать GitOps-деплой через Argo CD и подготовить `kustomize`-конфигурации для `dev` и `prod`.

## Исходные образы (Docker Hub)

Используйте готовые контейнерные образы:

- `mablinov2704/frontend:latest` - <https://hub.docker.com/r/mablinov2704/frontend>
- `mablinov2704/bff:latest` - <https://hub.docker.com/r/mablinov2704/bff>
- `mablinov2704/user-service:latest` - <https://hub.docker.com/r/mablinov2704/user-service>
- `mablinov2704/message-service:latest` - <https://hub.docker.com/r/mablinov2704/message-service>

Дополнительно допускается использование официальных образов:

- `postgres:16-alpine`
- `ghcr.io/kukymbr/goose-docker:latest` (для миграций)
- `minio/minio:latest` (если выбрано локальное S3-совместимое хранилище)

## Что нужно сделать

1. Развернуть в Kubernetes-кластере:
   - frontend
   - bff
   - user-service
   - message-service
   - postgres
   - миграции для `user-service` и `message-service`
2. Подключить S3-хранилище для загрузки файлов (из `message-service`) **через CSI-монтирование**.
3. Настроить правила `nodeAffinity` по условиям задания.
4. Подготовить `kustomize`-структуру для `dev` и `prod`.
5. Настроить Argo CD для автоматического деплоя из Git-репозитория.

## Краткие требования (выжимка из `docs`)

- **Архитектура в кластере:** frontend, bff, user-service, message-service, postgres и миграции должны запускаться как единая рабочая система.
- **S3 через CSI:** файловое хранилище для `message-service` подключается только через CSI-монтирование (MinIO или внешний S3-совместимый сервис).
- **`nodeAffinity`:**
  - `postgres` (и `minio`, если используется) размещать на `workload=system`;
  - прикладные сервисы размещать на `workload=app`;
  - для `message-service` обязательно: hard-условие `workload=app` + soft-предпочтение `disk=fast`.
- **`kustomize`:**
  - обязателен `base` и overlays `dev`/`prod`;
  - в `dev` и `prod` должны быть осмысленные различия (реплики, ресурсы, host, affinity, теги образов).
- **Argo CD (GitOps):**
  - `Application` должен смотреть на ваш GitHub-репозиторий и один из overlays;
  - автосинхронизация обязательна: `automated`, `prune`, `selfHeal`.
- **Проверка перед сдачей:** оба overlays собираются, Pods работают, загрузка файлов работает через S3 CSI, Argo CD в состоянии `Synced/Healthy`.

## Сервисы и обязательные env-переменные

Ниже приведены ключевые переменные окружения, которые должны быть корректно заданы в Kubernetes-конфигурации.

- **`web-ui` (frontend):**
  - `BFF_URL` - публичный URL API для браузера (может быть пустым при same-origin).
  - `BFF_INTERNAL_URL` - внутренний адрес API-шлюза внутри кластера.
- **`bff` (API-шлюз):**
  - `HTTP_PORT` - порт запуска сервиса.
  - `USER_SERVICE_URL` - внутренний URL сервиса пользователей.
  - `MSG_SERVICE_URL` - внутренний URL сервиса сообщений.
- **`user-service`:**
  - `HTTP_PORT` - порт запуска сервиса.
  - `DB_DSN` - строка подключения к БД пользователей.
- **`message-service`:**
  - `HTTP_PORT` - порт запуска сервиса.
  - `DB_DSN` - строка подключения к БД сообщений.
  - `UPLOADS_DIR` - путь до директории, смонтированной через S3 CSI.
- **`postgres`:**
  - `POSTGRES_USER` - пользователь БД.
  - `POSTGRES_PASSWORD` - пароль БД.
  - `POSTGRES_DB` - bootstrap-имя БД.
- **`migrate-users` / `migrate-messages` (jobs миграций):**
  - `GOOSE_DRIVER` - драйвер БД (`postgres`).
  - `GOOSE_DBSTRING` - строка подключения к целевой БД миграций.
  - `GOOSE_MIGRATION_DIR` - путь к SQL-миграциям в контейнере.

## Ограничения и требования

- Изменять исходный код сервисов не нужно.
- В рамках работы изменяются только Kubernetes/GitOps-конфигурации и инфраструктурные файлы.
- Все артефакты должны храниться в вашем GitHub-репозитории.
- Итоговая защита: ссылка на репозиторий с корректной структурой `kustomize` и рабочим Argo CD Application.

## Ожидаемая структура в вашем репозитории

Вы можете использовать любой удобный путь, но рекомендуется структура:

- `k8s/base/` - базовая конфигурация
- `k8s/overlays/dev/` - конфигурация dev
- `k8s/overlays/prod/` - конфигурация prod
- `argocd/` - Argo CD Application (и при желании AppProject)
- `docs/` - пояснения и скриншоты/результаты проверки

## Критерии приемки

Работа считается выполненной, если:

- все сервисы приложения доступны и корректно взаимодействуют;
- миграции применяются штатно;
- загрузка файлов в `message-service` работает через подключенное S3 CSI;
- реализованы требования по `nodeAffinity`;
- есть рабочие `kustomize`-overlay для `dev` и `prod`;
- Argo CD автоматически синхронизирует окружение из Git;
- в репозитории присутствуют все необходимые конфигурации и инструкция по запуску.

## Обязательные материалы в `docs`

Теория, примеры и шаблоны вынесены в папку `docs`:

- `docs/01-architecture-and-resources.md`
- `docs/02-k8s-manifests-examples.md`
- `docs/03-s3-csi.md`
- `docs/04-node-affinity-task.md`
- `docs/05-kustomize-task.md`
- `docs/06-argocd-task.md`
- `docs/07-checklist-and-defense.md`

Ориентируйтесь на эти документы как на техническое задание и справочник.
