# syntax=docker/dockerfile:1.7

FROM golang:1.22-alpine AS builder
WORKDIR /src

# Сертификаты на всякий случай, если захочу прикрутить когда-то TLS
RUN apk add --no-cache ca-certificates

COPY go.mod go.sum ./

RUN --mount=type=cache,target=/go/pkg/mod \
    go mod download

COPY . .

RUN --mount=type=cache,target=/go/pkg/mod \
    --mount=type=cache,target=/root/.cache/go-build \
    go test ./...

# Немного для кросс-компиляции
ARG TARGETOS=linux
ARG TARGETARCH

# Сборка приложения
RUN --mount=type=cache,target=/go/pkg/mod \
    --mount=type=cache,target=/root/.cache/go-build \
    CGO_ENABLED=0 \
    GOOS=${TARGETOS:-linux} \
    GOARCH=${TARGETARCH:-amd64} \
    go build \
      -trimpath \
      -ldflags="-s -w" \
      -o /out/momo-store-api \
      ./cmd/api

FROM alpine:3.20 AS runtime

# Создаю пользователя app, ставим сертификат, готовим папочки
RUN addgroup -g 10001 -S app \
    && adduser -u 10001 -S -G app -H -h /app app \
    && apk add --no-cache ca-certificates \
    && mkdir -p /app/data /tmp \
    && chown -R app:app /app /tmp \
    && rm -rf /var/cache/apk/*

WORKDIR /app

#копируем бинарник из билда в рантайм
COPY --from=builder --chown=app:app /out/momo-store-api /app/momo-store-api

ENV APP_ENV=production

# Запускаем контейнет от пользователя app
USER 10001:10001

# Приложение внутри слушает порт
EXPOSE 8081

# Чекап здоровья контейнера
HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
  CMD wget -q -O /dev/null http://127.0.0.1:8081/health || exit 1

# При старте контейнера запускам
ENTRYPOINT ["/app/momo-store-api"]