FROM node:22-bookworm
ENV DEBIAN_FRONTEND=noninteractive \
    HUSKY=0 \
    CI=true \
    PNPM_HOME=/pnpm \
    PATH=/pnpm:/usr/lib/postgresql/16/bin:$PATH \
    PGDATA=/var/lib/postgresql/16/scanner

# PostgreSQL 16 (the version production and the Testcontainers suites use) from the PGDG repo, plus Redis for
# the idempotency/BullMQ paths. There is no Docker in the scanner, so Testcontainers cannot start a database;
# a local cluster lets the agent apply the real migrations and probe the RLS policies directly.
RUN apt-get update && apt-get install -y --no-install-recommends ca-certificates curl gnupg git python3 build-essential \
    && install -d /usr/share/postgresql-common/pgdg \
    && curl -fsSL https://www.postgresql.org/media/keys/ACCC4CF8.asc -o /usr/share/postgresql-common/pgdg/apt.postgresql.org.asc \
    && echo "deb [signed-by=/usr/share/postgresql-common/pgdg/apt.postgresql.org.asc] https://apt.postgresql.org/pub/repos/apt bookworm-pgdg main" > /etc/apt/sources.list.d/pgdg.list \
    && apt-get update && apt-get install -y --no-install-recommends postgresql-16 postgresql-client-16 redis-server \
    && rm -rf /var/lib/apt/lists/*

# scanner contract: the checkout lives inside the image, at /src
COPY . /src
WORKDIR /src

# Same steps as CI (.github/workflows/ci.yml): pnpm via corepack, frozen install, compile every workspace package.
RUN corepack enable && corepack prepare pnpm@9.15.0 --activate
RUN pnpm install --frozen-lockfile
RUN pnpm -r run build

# A local cluster with every migration in packages/db/drizzle applied, as the superuser `postgres` (no password,
# trust auth on localhost). Start it with: su postgres -c "/usr/lib/postgresql/16/bin/pg_ctl -D $PGDATA -l /tmp/pg.log start"
RUN install -d -o postgres -g postgres "$PGDATA" \
    && su postgres -c "/usr/lib/postgresql/16/bin/initdb -D $PGDATA --auth=trust -U postgres" \
    && su postgres -c "/usr/lib/postgresql/16/bin/pg_ctl -D $PGDATA -l /tmp/pg.log -w start" \
    && createdb -U postgres -h 127.0.0.1 retail_tower \
    && DATABASE_URL=postgres://postgres@127.0.0.1:5432/retail_tower node packages/db/dist/cli/migrate.js up \
    && su postgres -c "/usr/lib/postgresql/16/bin/pg_ctl -D $PGDATA -w stop"

# Docker-free unit tests (the Testcontainers suites skip themselves). Failures do not abort the build.
ENV MIGRATION_TEST_ALLOW_SKIP=1
RUN pnpm --filter @data-pulse-2/auth --filter @data-pulse-2/shared --filter @data-pulse-2/worker test \
    || echo "WARNING: unit tests reported failures"
RUN pnpm --filter @data-pulse-2/api test -- --coverage=false --testPathPattern='\.unit\.spec\.ts$' \
    || echo "WARNING: api unit tests reported failures"
