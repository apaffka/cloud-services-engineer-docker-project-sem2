# syntax=docker/dockerfile:1.7

FROM node:16-alpine AS deps

WORKDIR /app

# Так кешируем установку зависимостей
COPY package.json package-lock.json ./

# Запускаем установку зависимостей
RUN --mount=type=cache,target=/root/.npm \
    npm ci --no-audit --no-fund

# Тут у нас уже будут зависимости из deps
FROM deps AS builder

WORKDIR /app

ARG VUE_APP_API_URL=/api
ENV VUE_APP_API_URL=${VUE_APP_API_URL}
ENV NODE_ENV=production

COPY . .

# Сборка фронтенда
RUN npm run build

# Образ для запуска без root
FROM nginxinc/nginx-unprivileged:1.27-alpine AS runtime

COPY --chown=101:101 nginx.conf /etc/nginx/conf.d/default.conf
# Копирование собранного приложения
COPY --from=builder --chown=101:101 /app/dist /usr/share/nginx/html/momo-store

# Запуск контейнера от пользователя 101
USER 101:101

# Приложение слушает порт
EXPOSE 8080

# Чекап здоровья контейнера
HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
  CMD wget -q -O /dev/null http://127.0.0.1:8080/health || exit 1