# D11-01 — Metabase 0.65.2, vendored definition

| Field | Value |
| --- | --- |
| Product | Metabase 0.65.2 (`metabase/metabase`, GitHub release `v0.65.2`, 2026-10-30) |
| Image | `metabase/metabase:v0.65.2@sha256:b62ac148…518a9e` |
| Sidecar | `postgres:18.7-alpine@sha256:0115267a…247e71`, service `db` |
| Port | `3000` (`httpPort`), written as `expose: ['3000']`; no host port |
| Identity | `fa_gate` — forward-auth gate, product keeps its own login form |
| Volume | `mb-pgdata` → `/var/lib/postgresql` |
| Resources | 2 vCPU / 4 GiB / 20 GiB |

## Why this tree exists

`metabase/metabase` ships a **build** Dockerfile (multi-stage Clojure/Java + Yarn compile) and no
Compose file. The executor's `docker buildx build` needs a BuildKit session the scoped socket proxy
refuses (403), so the `dockerfile` path died at `apply`. The project's published artefact is the image
`metabase/metabase`, which is what this tree pins.

## Why PostgreSQL and not the embedded H2

The image will start with no database and keep its application data in an H2 file under
`/metabase-data`. Upstream restricts that to evaluation: it serialises writes, corrupts under
concurrent access and has no repair path. Saved questions, dashboards and users therefore live in the
pinned PostgreSQL sidecar, and the entry's `database` dependency is served inside this definition.

`mb-pgdata` is mounted at `/var/lib/postgresql`, the PostgreSQL 18 image's declared VOLUME. That image
keeps one data directory per major version (`/var/lib/postgresql/18/docker`), so mounting
`/var/lib/postgresql/data` would leave the cluster outside the mount.

## Environment (`catalog/env/D11-01.json`)

| Variable | Purpose |
| --- | --- |
| `MB_SITE_URL` | Published route; every generated link (resets, subscriptions, alerts) is built from it. The image otherwise falls back to the forwarded `Host` header. |
| `MB_DB_HOST` / `_PORT` / `_DBNAME` / `_USER` | Fixed to `db:5432`, database and role `metabase`; the sidecar derives `POSTGRES_DB` / `POSTGRES_USER` from the same values. Read once — the role is created only while the data directory is empty. |
| `MB_DB_PASS` | Required secret; also the sidecar's `POSTGRES_PASSWORD`. Without it Metabase silently falls back to H2. |
| `MB_ENCRYPTION_SECRET_KEY` | Required, exactly 64 hex characters. Without it every data-source credential is stored in plaintext; rotating it makes stored credentials undecryptable. |
| `MB_SESSION_SECRET_KEY` | Required, exactly 64 hex characters. Without it session keys are unsigned and anyone with read access to the application database can forge a session. |
| `MB_ANON_TRACKING_ENABLED` | Default `false` — the free edition pings upstream otherwise. |
| `MB_LOAD_SAMPLE_CONTENT` | Default `false`. First boot would otherwise install an `Examples` database and ~40 demo cards. Read once; it removes nothing already installed. |

## Operational notes

- **First boot is slow.** The sidecar initialises, then Metabase migrates its schema before
  `/api/health` answers `200`. Healthcheck `start_period` is 300 s; `verify` waits
  `SEAMNEX_VERIFY_TIMEOUT_MS`.
- **`/api/health` is the readiness signal.** `/` answers `200` with the loading page while migrations
  run, so a status-below-500 probe can report `ready` too early; the container health status is what
  says the product is up.
- **First run opens the setup wizard.** `GET /` redirects to `/setup` until an administrator exists.
  That is business configuration, not a deployment defect.
- **Free edition.** JWT/SAML SSO, sandboxing, audit logs and API keys are Pro/Enterprise features, so
  the open-source edition keeps its own sign-in behind the gate; `sloSupported` is `false` and
  `integration.authMethod` brokers a session per user via `POST /api/session`. The gap is the entry's
  recorded `deviation`.

## Verified behaviour

| Probe | Result |
| --- | --- |
| `GET /api/health` inside the container | `200`, `{"status":"ok"}` |
| `GET /` at the edge with a platform session | `200`, setup page |
| `GET /` at the edge without a platform session | `401` (forward-auth) |
| Boot log | `Saved credentials encryption is ENABLED`; no "session keys … without a signature" warning |
| Application database | PostgreSQL sidecar, schema migrated, cluster in `mb-pgdata` |

## Verification notes

Reviewed against `catalog/D11-01.json` and `catalog/env/D11-01.json` on 2026-11-26.

- **Removed:** the earlier instruction to supply the secrets with `--env-file temp/d11.env`. The env
  schema is the authority: all three secrets are declared `required` and resolved from the platform
  secret store at apply time.
- **Corrected:** the earlier probe row said `postgres:18-alpine`; the definition pins
  `postgres:18.7-alpine` by tag and digest.
- **Not in the catalog JSON:** the H2 corruption rationale beyond the free-text `notes`, the 300 s
  healthcheck start period, and the setup-wizard behaviour. They describe the image and the Compose
  file, which the catalog only summarizes.
- `sessionMaxMinutes: 480` is platform policy, not the product's cookie lifetime (the record's
  `deviation` says so); no declared variable enforces it.
