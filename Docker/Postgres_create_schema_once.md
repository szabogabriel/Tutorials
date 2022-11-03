# Postgres - create image with pre-built schema

This is a 'backup' of the article found [here](https://cadu.dev/creating-a-docker-image-with-database-preloaded/).

```Dockerfile
FROM postgres:11-alpine as dumper

COPY test_dump.sql /docker-entrypoint-initdb.d/

RUN ["sed", "-i", "s/exec \"$@\"/echo \"skipping...\"/", "/usr/local/bin/docker-entrypoint.sh"]

ENV POSTGRES_USER=postgres
ENV POSTGRES_PASSWORD=postgres
ENV PGDATA=/data

RUN ["/usr/local/bin/docker-entrypoint.sh", "postgres"]

# final build stage
FROM postgres:11-alpine

COPY --from=dumper /data $PGDATA
```
## Simple dockerfile

Example taken from [here](https://dev.to/andre347/how-to-easily-create-a-postgres-database-in-docker-4moj)

```Dockerfile
FROM postgres
ENV POSTGRES_PASSWORD docker
ENV POSTGRES_DB world
COPY world.sql /docker-entrypoint-initdb.d/
```

## Docker compose

```Yaml
version: '3.1'

services:

  db:
    image: postgres:10
    restart: unless-stopped
    environment:
      POSTGRES_PASSWORD: db-password
      POSTGRES_USER: db-user
      POSTGRES_DB: db-name
    volumes:
      - /home/user/postgres-volume
    ports:
      - "6432:5432"
```
