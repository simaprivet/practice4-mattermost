# Образ с официальной утилитой Mattermost migration-assist
# (https://github.com/mattermost/migration-assist).
# Собирается из исходников, поэтому работает и на amd64, и на arm64.
FROM golang:1.24-alpine AS build
ARG MIGRATION_ASSIST_VERSION=v0.8
# Медленная/нестабильная сеть: несколько зеркал Go-модулей.
# Разделитель "|" — переход на следующее зеркало при ЛЮБОЙ ошибке (а не только 404).
ENV GOPROXY="https://proxy.golang.org|https://goproxy.io|https://goproxy.cn|direct" \
    GOSUMDB=off \
    GOFLAGS=-mod=mod \
    CGO_ENABLED=0
# До 3 попыток: при обрыве сети скачивание продолжится с уже загруженных модулей
RUN for i in 1 2 3; do \
      go install github.com/mattermost/migration-assist/cmd/migration-assist@${MIGRATION_ASSIST_VERSION} && exit 0; \
      echo "go install failed (attempt $i), retrying..."; sleep 10; \
    done; exit 1

FROM alpine:3.20
COPY --from=build /go/bin/migration-assist /usr/local/bin/migration-assist
WORKDIR /work
CMD ["migration-assist", "--help"]
