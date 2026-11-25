# Образ с официальной утилитой Mattermost migration-assist v0.8
# (https://github.com/mattermost/migration-assist).
#
# Готовая сборка из официального релиза лежит рядом в репозитории:
#   migration-assist-Linux-x86_64.tar.gz
#   (https://github.com/mattermost/migration-assist/releases/download/v0.8/migration-assist-Linux-x86_64.tar.gz)
# Поэтому при сборке образа НИЧЕГО не скачивается из интернета:
# сборка из исходников (go install) падала из-за недоступности Go-зеркал.
# Целостность архива проверяется по SHA-256.
FROM alpine:3.20
COPY migration-assist-Linux-x86_64.tar.gz /tmp/ma.tar.gz
RUN echo "d17e41e38762e79c17324da0dd10dcb9fd0755d97fecd5315bfabbb295b7211b  /tmp/ma.tar.gz" | sha256sum -c - \
 && tar -xzf /tmp/ma.tar.gz -C /usr/local/bin migration-assist \
 && chmod +x /usr/local/bin/migration-assist \
 && rm /tmp/ma.tar.gz \
 && migration-assist version
WORKDIR /work
CMD ["migration-assist", "--help"]
