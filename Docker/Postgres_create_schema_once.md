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
