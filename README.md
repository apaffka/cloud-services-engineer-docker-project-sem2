# Momo Store by Agapov

## Состав
- `backend/Dockerfile` — multi-stage сборка Go backend.
- `frontend/Dockerfile` — multi-stage сборка Vue frontend и запуск через unprivileged Nginx.
- `frontend/nginx.conf` — раздача SPA и reverse proxy `/api/*` на backend.
- `docker-compose.yml` — запуск сервисов, сети, volumes, healthchecks, лимиты и security.
- `.env.example` — пример переменных окружения.
- `.github/workflows/deploy.yaml` — CI/CD pipeline со сборкой, Trivy и push в DockerHub.

## Docker images
Backend собирается в два этапа:
1. `golang:1.26.4-alpine` — сборка и тестирование Go.
2. `alpine:3.23.4` — минимальный runtime, куда копируется только готовый бинарник.

Frontend также собирается в два этапа:
1. `node:16-alpine` — установка зависимостей и сборка Vue.
2. `nginxinc/nginx-unprivileged:1.31.1-alpine3.23` — runtime для раздачи статических файлов без root.

Финальные образы не содержат Go compiler, Node.js build tools и исходные зависимости.

## Быстрый запуск

Перед первым запуском подготовить `.env` и secret-файл:

```bash
cp .env.example .env
mkdir -p secrets
touch secrets/.gitkeep
openssl rand -base64 32 > secrets/backend_app_secret.txt
chmod 600 secrets/backend_app_secret.txt
```

Production profile:

```bash
COMPOSE_PROFILES=prod docker compose up --build -d
```

Открыть приложение:

```text
http://localhost/momo-store/
```

Проверка healthcheck и API:

```bash
curl http://localhost/health
curl http://localhost/api/products
curl http://localhost/api/categories
```

Остановка production profile:

```bash
COMPOSE_PROFILES=prod docker compose down --remove-orphans
```

## Development запуск

Dev profile публикует backend напрямую на `localhost:8081`, чтобы его можно было проверять отдельно.

```bash
COMPOSE_PROFILES=dev docker compose up --build -d
```

Проверка:

```bash
curl http://localhost:8081/health
curl http://localhost:8081/products
curl http://localhost:8080/health
```

Frontend в dev-режиме:

```text
http://localhost:8080/momo-store/
```

Остановка dev profile:

```bash
COMPOSE_PROFILES=dev docker compose down --remove-orphans
```

## Docker Compose

В `docker-compose.yml` настроены два профиля:

- `prod` — production запуск;
- `dev` — debug запуск с публикацией backend-порта на localhost.

В production наружу публикуется только frontend:

```text
host port 80 -> frontend container port 8080
```

Backend не публикуется напрямую наружу и доступен только внутри Docker-сети по имени:

```text
backend:8081
```

Frontend Nginx проксирует API-запросы:

```text
/api/* -> backend:8081
```

## Volumes

Настроены volumes:

- `momo-store-backend-data` -> `/app/data`;
- `momo-store-frontend-cache` -> `/var/cache/nginx`.

Backend использует fake store, поэтому volume подготовлен и не хранит критичных данных.

## Healthchecks

Healthchecks настроены для всех сервисов:

- backend: `http://127.0.0.1:8081/health`;
- frontend: `http://127.0.0.1:8080/health`.


## Масштабирование

Backend можно масштабировать горизонтально:

```bash
COMPOSE_PROFILES=prod docker compose up --build -d --scale backend=3
```

Проверка:

```bash
docker compose ps
curl http://localhost/api/products
```

## Security hardening

В контейнерах настроены базовые и расширенные меры безопасности:

- контейнеры запускаются не от root: backend `10001:10001`, frontend `101:101`;
- включён `read_only: true`;
- временные директории вынесены в `tmpfs`;
- Linux capabilities отключены через `cap_drop: ALL`;
- запрещено повышение привилегий через `no-new-privileges:true`;
- заданы лимиты CPU, memory и PID;
- наружу открыт только необходимый порт frontend;
- backend доступен только во внутренней сети в production;
- Docker Secrets используются для подключения secret-файла;
- `.dockerignore` исключает ненужные файлы из build context.

## Docker Secrets

Secret создаётся локально и подключается в Compose:

```bash
openssl rand -base64 32 > secrets/backend_app_secret.txt
chmod 600 secrets/backend_app_secret.txt
```

Файл secret не должен попадать в git. Заглушка:

```text
secrets/.gitkeep
```

## CI/CD

В pipeline `.github/workflows/deploy.yaml` добавляем проверки безопасности.

Основные шаги:
1. checkout repository;
2. setup Docker Buildx;
3. build backend image locally for scan;
4. build frontend image locally for scan;
5. scan backend image with Trivy;
6. scan frontend image with Trivy;
7. login to DockerHub;
8. push backend image to DockerHub;
9. push frontend image to DockerHub;
10. build and run application via Docker Compose;
11. smoke-test через `curl`.

Trivy scan настроен на проверку `HIGH` и `CRITICAL` уязвимостей.

## DockerHub images

После успешного pipeline публикуются образы:

```text
<DOCKER_USER>/docker-project-backend:latest
<DOCKER_USER>/docker-project-frontend:latest
```

`DOCKER_USER` и `DOCKER_PASSWORD` настроены в GitHub Secrets.