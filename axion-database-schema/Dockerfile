# syntax=docker/dockerfile:1

#############################################################
# Stage 1 - Validate the SQL migrations against a scratch DB.
# Catches syntax/DDL errors at build time, before shipping.
#############################################################
FROM postgres:17-alpine AS validator

WORKDIR /work

COPY *.sql /work/

USER postgres

RUN initdb -D /tmp/pgdata --auth=trust -U postgres && \
    mkdir -p /tmp/pgsock && \
    pg_ctl -D /tmp/pgdata -l /tmp/pg.log -o "-p 5433 -k /tmp/pgsock" start && \
    for i in $(seq 1 30); do \
      pg_isready -h /tmp/pgsock -p 5433 -U postgres >/dev/null 2>&1 && break; \
      sleep 0.2; \
    done && \
    pg_isready -h /tmp/pgsock -p 5433 -U postgres && \
    psql -h /tmp/pgsock -p 5433 -U postgres -d postgres -v ON_ERROR_STOP=1 -f /work/01-extensions.sql && \
    psql -h /tmp/pgsock -p 5433 -U postgres -d postgres -v ON_ERROR_STOP=1 -f /work/02-telemetry.sql && \
    pg_ctl -D /tmp/pgdata stop

#############################################################
# Stage 2 - Runtime ships the validated SQL as initdb scripts.
# Used standalone to boot a fresh DB, or (in K8s) from an
# initContainer that seeds the Postgres data dir.
#############################################################
FROM postgres:17-alpine AS runtime

COPY --from=validator /work/*.sql /docker-entrypoint-initdb.d/

EXPOSE 5432

# Note: keep default (root) entrypoint user - the official postgres
# docker-entrypoint.sh chowns PGDATA and drops to the postgres user itself
# when it starts the server, so a non-root image USER would break fresh boots.
HEALTHCHECK --interval=30s --timeout=5s --start-period=30s --retries=5 \
  CMD ["pg_isready", "-U", "postgres"]

CMD ["postgres"]