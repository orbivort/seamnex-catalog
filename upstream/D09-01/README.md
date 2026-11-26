# D09-01 — OpenProject 17.9.6, vendored definition

| Field | Value |
| --- | --- |
| Product | OpenProject 17.9.6 (`openproject/openproject`, GitHub release `v17.9.6`, 2026-10-30) |
| Image | `openproject/openproject:17.9.6@sha256:c4802aca…3fdff98` |
| Port | `80` (`httpPort`) — the container port, written as `expose: ['80']`; no host port |
| Identity | `fa_gate` — forward-edge gate, product shows its own sign-in |
| Volumes | `pgdata` → `/var/openproject/pgdata`, `assets` → `/var/openproject/assets` |
| Resources | 4 vCPU / 4 GiB / 50 GiB (the two volume quotas: 30 GiB + 20 GiB) |

## Why this tree exists

`opf/openproject` ships a **build** Dockerfile (`docker/prod/Dockerfile`) and no Compose file. The
executor's `docker compose build` needs a BuildKit session the scoped socket proxy refuses, so the job
reached `apply` and died with `unable to upgrade to tcp, received 403`. The published all-in-one image
is what production deployments use, and it is what this tree pins.

## What the image contains

One container under supervisord: PostgreSQL 17 (`$PGDATA=/var/openproject/pgdata`), memcached, the
Rails/Puma application and its worker, and Apache on port 80 (`ServerName` from `SERVER_NAME`).
Because the database, cache and worker live inside the container, the entry's `database`, `cache` and
`worker` dependency roles are satisfied internally — do not add services for them.

The application tree (`/app`) is deliberately not mounted; the image owns it.

## Environment (`catalog/env/D09-01.json`)

| Variable | Purpose |
| --- | --- |
| `OPENPROJECT_HOST__NAME` / `SERVER_NAME` | Route host, as a **bare host** (no scheme, no port), and identical in both. A URL-shaped value is copied into Rails' host allowlist and every request answers `400 Invalid host_name configuration`. |
| `OPENPROJECT_HTTPS` / `OPENPROJECT_HSTS` | `true` / `false`. TLS ends at the edge, so Rails must build `https` links and mark cookies `Secure`, but must not emit its own HSTS header or redirect the internal plain-HTTP hop the relay's admin client uses. |
| `SECRET_KEY_BASE` | Required secret. The image ships `OVERWRITE_ME`; inheriting it makes session cookies forgeable. Rotating it signs everyone out. |
| `OPENPROJECT_SEED__ADMIN__USER__PASSWORD` | Required secret, read once while no administrator exists. Without it the seeder applies its documented `admin` default. |
| `OPENPROJECT_SEED__ADMIN__USER__PASSWORD__RESET` | Default `false`. Note the doubled underscores: the single-underscore spelling is silently ignored. Read at first boot only; an instance seeded with `true` keeps owing the change. |

## Operational notes

- **First boot is slow.** The entrypoint creates the cluster, migrates and seeds before Apache
  answers. The healthcheck's `start_period` is 900 s; `verify` waits `SEAMNEX_VERIFY_TIMEOUT_MS`.
- **Apache answers a 200 placeholder before Rails is up.** A host-side `GET /` can therefore see
  `ready` while migrations still run. The container health status is the trustworthy signal.
- **Healthcheck before readiness.** `/health_checks/default` is OkComputer's own check; the probe sends
  `Host: $SERVER_NAME` (without it, Apache's `_default_` vhost answers 404) and asserts a literal
  `200` (`curl -f` accepts a 3xx).
- **Sizing is the all-in-one minimum** — PostgreSQL, Puma, Apache and the worker share one cgroup;
  1 vCPU / 2 GiB risks an OOM during first-boot migrations.
- **Community edition.** Native OIDC/SAML and several board features need an Enterprise token. The
  OmniAuth strategy is filtered at boot (`EnterpriseToken.allows_to?(:sso_auth_providers)`), the
  product consumes no forwarded identity header, and the registered identity client is unused. The
  gap is recorded as the entry's `deviation`, not hidden; `sloSupported` is `false`.
- **Still missing:** no SMTP configuration (invitations and password resets are not delivered), and
  only `docker/compose.dev.yml` publishes a development edge.

## Verified behaviour

| Probe | Result |
| --- | --- |
| `GET /health_checks/default` with the route `Host` | `200`, `Application is running` |
| `GET /login` | `200`, `Sign in | OpenProject` |
| Sign-in as the seeded `admin` | forced password change, then the home page |
| `GET /api/v3/work_packages` with a session | `200`, `WorkPackageCollection` |
| `GET /` at the edge without a platform session | `401` (forward-auth) |

The route is served over TLS (`https://<host>:8443` in development, `443` in production). Older
measurements in this file were taken on the removed plain-HTTP `:8020` edge and are historical.

## Verification notes

Reviewed against `catalog/D09-01.json` and `catalog/env/D09-01.json` on 2026-11-26.

- **Corrected paths:** the environment declaration is `catalog/env/D09-01.json` (earlier text pointed
  at `platform/catalog/env/…`). The digest and `httpPort: 80` in the table match the catalog record.
- **Not in the catalog JSON:** "PostgreSQL 17 + memcached + worker inside the image", the 900 s
  healthcheck start period, and the 200-placeholder behaviour. They describe the image and the
  Compose file, which the catalog summarizes only as a single-service definition.
- **Not in the catalog JSON:** the Enterprise gating evidence. The catalog records the resulting
  `fa_gate` class and the deviation text, not the measured `EnterpriseToken` result.
- The dated deployment records (2026-10-31 to 2026-11-04, `:8020`) were removed: that edge no longer
  exists and the values cannot be verified against anything current.
