# Threat model

## What this project does and where untrusted input enters
Backend-Core is the multi-tenant SaaS backend of Retail Tower, a retail/pharmacy point-of-sale platform. It is a
NestJS 11 + PostgreSQL 16 (row-level security) + Redis/BullMQ pnpm monorepo:

- `apps/api` — the HTTP API. Every route is reachable from the internet behind a reverse proxy.
- `apps/worker` — BullMQ consumers (outbox drainer, ERP posting/reconciliation, audit, retention sweeps).
- `packages/auth` (password and token hashing), `packages/db` (schema, SQL migrations, tenant-context
  middleware, migration CLI), `packages/shared`, `packages/contracts/openapi` (the OpenAPI 3.1 contracts).

Three kinds of remote client send untrusted input, and each must stay inside its own tenant (and store):

- **Dashboard users** (browser): `api/v1/{auth,context,tenants,stores,memberships,audit,catalog/*,settlement/*,admin/outbox}`
  and `api/inventory/v1/*`. Authenticated by the `dp2_session` httpOnly cookie or an opaque bearer token.
- **POS terminals** (Windows desktop app in a shop): `api/pos/v1/{terminals/pair,operators,cashier-admissions,catalog,sales,shifts,audit-events}`.
  A paired device holds a device token; a signed-in operator holds a short-lived `pos_operator` envelope token.
  A terminal sits in a shop and should be treated as only partly trusted: a stolen device token must not reach
  another tenant or store, or act as an operator.
- **The ERPNext connector** (service-to-service): `api/v1/connector/*` and `api/connector/v1/erpnext[/health]`,
  with a `connector`-scope token tied to one tenant's `connector_registration`. Data it pushes back (posting
  outcomes, ERPNext bin/stock reports) is stored and later shown to dashboard users.

Unauthenticated routes: sign-in, password-reset request/confirm, email-verify confirm (`apps/api/src/auth/auth.controller.ts`),
health (`apps/api/src/health`), and `POST api/pos/v1/terminals/pair` (pairing-code exchange).

## How the security boundaries work (what to try to break)
- **Fail-closed authentication.** `apps/api/src/auth/fail-closed-auth.guard.ts` is the global `APP_GUARD`. A route
  is authenticated unless it carries one of the markers in `apps/api/src/auth/route-auth.ts`. Any route that is
  reachable with less authentication than intended is in scope.
- **Credentials.** Session cookies, bearer tokens, device tokens and pairing codes are random values stored only as
  SHA-256 hashes and compared in constant time (`packages/auth/src/tokens.ts`). Passwords use argon2id
  (`packages/auth/src/passwords.ts`). Token scopes are `dashboard_api`, `pos`, `pos_operator`, `connector`, plus
  single-use `password_reset` / `email_verify`, which must never work as API credentials.
- **POS auth.** `pos-device-auth.guard.ts` (tenant and store come only from the device row),
  `pos-operators/*` (operator sign-in verifies a Clerk JWT in `clerk-verifier.ts`, then issues the envelope),
  `catalog/sales/sale-capture-auth.guard.ts` (chooses the envelope path or the device-plus-cashier-admission path),
  `pos-shifts/shift-cash-up-auth.guard.ts`. Confusion between these paths (for example, a device token that is
  accepted where an operator is required, or an `operatorUserId` in a body that is trusted without an admission)
  is in scope.
- **Tenant isolation is the central invariant.** `apps/api/src/context/tenant-context.guard.ts` resolves the tenant
  (and active store) from the session or the token binding; `packages/db/src/middleware/tenant-context.ts` opens a
  transaction and sets `app.current_tenant` with `set_config(..., true)`; tables use FORCE ROW LEVEL SECURITY
  policies defined in `packages/db/drizzle/*.sql`. The application role is non-superuser and NOBYPASSRLS
  (`docs/operations/database-roles.md`); pre-tenant lookups use a separate `AUTH_LOOKUP_DATABASE_URL` pool
  (`apps/api/src/auth/database-pools.ts`). Look for any query that runs outside the tenant transaction, a context
  value taken from request input, a policy that is missing or wrong on a tenant-owned table, or a path that lets
  one tenant read or write another tenant's rows, or one store's user act on another store.
- **Authorization.** `apps/api/src/auth/roles.guard.ts` denies by default; roles are `owner`, `tenant_admin`,
  `store_manager`, `store_staff`; the platform-admin bypass is for sessions only. Privilege escalation between
  these roles (for example, through membership management in `apps/api/src/memberships`) is in scope.
- **Idempotency.** `apps/api/src/idempotency/` (Redis fast path, Postgres mirror). A replayed response that leaks
  across clients or tenants, or a key collision that lets one client get another's stored response, is in scope.
- **Money.** Amounts are exact decimals (`numeric(19,4)` strings, quantities `numeric(19,6)`), never floats. Sale
  capture (`apps/api/src/catalog/sales/`), returns, shift cash-up and settlement/receivables
  (`apps/api/src/settlement/`) enforce invariants: tenders must equal the sale total, refunds must equal the server's
  computed total, line amounts must equal unit price × quantity, amounts are non-negative, settlement applications
  are positive. Capture is deduplicated on `(tenant_id, source_system, external_id)`. A way to record a sale,
  refund or payment application that breaks these, or to apply one tenant's money to another tenant's receivable,
  is in scope. Note that `pos_total` is stored as sent by the POS and a mismatch with the line sum only sets the
  advisory `mismatch_flag` — that is by design.
- **Validation.** Input is validated with Zod (`apps/api/src/common/zod-validation.pipe.ts` and per-route DTOs)
  against the contracts in `packages/contracts/openapi/**`. There are no file uploads.

## Components that matter most / least
- Most: `apps/api/src/auth`, `apps/api/src/context`, `packages/db/src/middleware`, `packages/db/src/helpers`,
  `packages/db/drizzle` (RLS policies and grants), `packages/auth`, `apps/api/src/pos-*`,
  `apps/api/src/catalog/sales`, `apps/api/src/settlement`, `apps/api/src/memberships`, `apps/api/src/connector`,
  `apps/api/src/idempotency`.
- Less: `apps/worker` (consumes jobs produced by the API, not by clients directly — but data stored from connector
  reports and rendered back to users is still in scope), `apps/api/src/observability`, `apps/api/src/health`.
- Out of scope: `loadtests/`, `scripts/`, `deploy/`, `docs/`, `specs/`, `tools/`, `.specify/`, `docker-compose*`,
  `Caddyfile`, the test suites under `apps/*/test` and `packages/*/__tests__`, and vulnerabilities that live only in
  third-party dependencies unless this code makes them reachable. The email adapter is a no-op in this repository.

## How to exercise it
- Everything is built in `/src` (`pnpm -r run build`; output in each package's `dist/`).
- Unit tests that need no database: `MIGRATION_TEST_ALLOW_SKIP=1 pnpm --filter @data-pulse-2/api test -- --coverage=false`
  (or one file: `pnpm --filter @data-pulse-2/api test -- --coverage=false test/auth/fail-closed-auth.guard.unit.spec.ts`).
  `auth`, `shared` and `worker` tests also run without a database. The integration suites start Postgres through
  Testcontainers, which needs Docker and will skip itself in this image.
- A PostgreSQL 16 cluster with every migration already applied is in the image (database `retail_tower`, superuser
  `postgres`, trust auth on localhost). Start it with
  `su postgres -c "/usr/lib/postgresql/16/bin/pg_ctl -D $PGDATA -l /tmp/pg.log start"` and connect with `psql -U postgres -h 127.0.0.1 retail_tower`.
  To test RLS as the application would, create a non-superuser NOBYPASSRLS role, grant it table privileges, and run
  queries inside `BEGIN; SELECT set_config('app.current_tenant', '<uuid>', true); ...`. `redis-server` is installed
  for the idempotency and queue paths.
- The API can run against that database: `DATABASE_URL=... AUTH_LOOKUP_DATABASE_URL=... REDIS_URL=redis://127.0.0.1:6379 node apps/api/dist/main.js`
  (see `apps/api/src/auth/database-pools.ts` and `docs/operations/database-roles.md` for the variables; the boot checks refuse a
  superuser `DATABASE_URL`).

## How you rate severity
- Critical: reading or writing another tenant's data; authentication bypass on any non-public route; using a
  device, connector or single-use token as a dashboard or operator credential; remote code execution; SQL injection.
- High: crossing stores inside a tenant against store-access rules; privilege escalation between roles; recording a
  sale, refund, cash-up or settlement application that breaks the money invariants above; a stolen device token
  acting as an operator; idempotency replay leaking another client's response; account takeover through the
  password-reset or email-verify flow.
- Medium: information disclosure without cross-tenant data (error details, enumeration of users or tenants), stored
  content from connector reports rendered unsafely, denial of service from a single request.
- Low: issues that need a misconfigured deployment (for example, a superuser `DATABASE_URL`, which the boot checks
  refuse) or an already-compromised admin.

## Anything to leave alone
- The advisory `pos_total` / `mismatch_flag` behaviour described above is intended.
- Settlement tax handling is deliberately inactive in v1; do not report missing VAT calculation.
- Do not report denial of service that only needs sustained traffic; rate limiting is done at the reverse proxy.
- Reports should include a reproduction against the code in this image (a unit test, a `psql` session against the
  migrated database, or HTTP requests against a locally started API) and a patch where possible.
