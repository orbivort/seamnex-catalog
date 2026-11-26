# D06-01 — ERPNext 16.53.0, vendored upstream tree

| Field | Value |
| --- | --- |
| Product | ERPNext 16.53.0 — <https://github.com/frappe/erpnext> (application only) |
| Packaging reference | <https://github.com/frappe/frappe_docker> `compose.yaml`, `compose.mariadb.yaml`, `compose.redis.yaml` |
| Image | `frappe/erpnext:v16.53.0@sha256:b961d69e…e9be2` — multi-arch, pinned by tag and index digest |
| Port | `8080` (`httpPort`, nginx via `NGINX_LISTEN_PORT`) |
| Identity | `native` — OIDC through Frappe's `Social Login Key` with the `Keycloak` provider |
| Shared with | `D05-01` (ERPNext Accounts) runs on this same deployment |
| Volumes | `sites` (20 GiB), `db-data` (40 GiB) — the 80 GiB reservation covers both |
| Resources | 4 vCPU / 8 GiB |

## Why this tree exists

`frappe/erpnext` ships no Compose file at any tag, so a clone-based resolution ends in an empty tree
and the deployment fails at `resolve`. The definition therefore lives here; the resolver copies it
into `<SEAMNEX_INSTANCE_DATA_DIR>/<id>/source/<ts>/` and reads `docker-compose.yml` from it.
`SOURCE_REVISION` is what the instance records as `source_revision`.

## Services

| Service | Role |
| --- | --- |
| `erpnext` | nginx on `8080`; carries the instance network alias `<slug>` and is what the route and readiness probe address. |
| `erpnext-backend` | gunicorn `:8000` plus the first-boot bootstrap; healthcheck is the `sites/<site>/.seamnex-ready` marker. |
| `erpnext-websocket` | socket.io `:9000`. Required: nginx declares it as an upstream and refuses to start if it does not resolve. |
| `erpnext-worker-short` / `erpnext-worker-long` | RQ workers for email, reports and imports. |
| `erpnext-scheduler` | `bench schedule`; a long-running loop, so it satisfies the "all containers running" rule. |
| `erpnext-db` | MariaDB 11.4.13 with `utf8mb4` / `utf8mb4_unicode_ci`. |
| `erpnext-redis-cache` / `erpnext-redis-queue` | Redis 7.4.12-alpine, no persistence. |

## Deviations from the reference layout

1. **No `configurator` service.** The platform requires every container to be `running`, so a
   one-shot container that exits makes `verify` time out. The backend's start-up script writes
   `common_site_config.json` instead.
2. **`bench new-site` is part of the definition.** Left manual, a fresh deployment serves nginx in
   front of an empty `sites` volume and answers "site not found" while reporting `ready`.
3. **`FRAPPE_SITE_NAME_HEADER` is a literal site name** (`ERPNEXT_SITE_NAME`), because the image uses
   it as `server_name`, as the `X-Frappe-Site-Name` header and as the `try_files` prefix. The URLs the
   product prints come from `ERPNEXT_SITE_ORIGIN` (`host_name`), re-applied on every start.
4. **No `pull_policy: always`, no `platform:` pin** — both are host-policy decisions the platform owns.

## First start

```text
mariadb healthy
      ▼
erpnext-backend
      ├─ common_site_config.json (db, redis, socketio port)
      ├─ bench new-site --install-app erpnext      (several minutes)
      ├─ sites/currentsite.txt, .seamnex-ready
      ├─ host_name <- ERPNEXT_SITE_ORIGIN          (every start)
      ├─ database account converged for the current container address
      ├─ Social Login Key <- ERPNEXT_SSO_*         (every start; skipped without a secret)
      └─ exec start.sh                             (gunicorn :8000)
      ▼
erpnext-websocket · workers · scheduler
      ▼
erpnext (nginx :8080)
```

A cold install can exceed the platform's 300 s default verify window; the host sets
`SEAMNEX_VERIFY_TIMEOUT_MS=1200000` before the first deployment. A reconfigure reuses the marker.

## Single sign-on

Frappe keeps identity configuration in a **site-database row**, not in the environment. The last step
of the backend bootstrap therefore opens a site context, upserts the `Social Login Key`, applies
`System Settings.session_expiry` / `disable_user_pass_login` and `Portal Settings.default_role`, and
commits. It runs on every start, which is what makes `redirect_url` follow the route; it is skipped
when `ERPNEXT_SSO_ENABLED` is false or no client secret exists. `bench execute --kwargs` was rejected
because the secret would appear in a process listing.

`ERPNEXT_SSO_CLIENT_SECRET` must be present **before** the container is created: the platform mints
and stores it before the render set is produced (`ensureIdentitySecret`), so the first boot already
has it.

### What the product supports

| Capability | Status |
| --- | --- |
| OAuth 2.0 / OIDC | native, per site (`Social Login Key`) |
| Keycloak as a first-class provider | yes (`login_via_keycloak`) |
| SAML 2.0, OIDC discovery, PKCE | no |
| ID-token signature verification | no — the userinfo call is the trust anchor |
| Claim → role mapping | no — only `Portal Settings.default_role`, on creation |
| Back-channel / RP-initiated logout | no |

The document must be **named** `keycloak` (`autoname` of `provider_name`, and the callback is
hardcoded to that key); a row left by an earlier name is deleted, because `provider_name` and
`social_login_provider` are `set_only_once`. The site database account is recreated on every start
for the container's current address, because `bench new-site` binds it to the address it ran from.

### Registered client and site row

| Keycloak client | Value |
| --- | --- |
| client id | `seamnex-<slug>` (derived; the site stores it as `client_id`) |
| protocol | `openid-connect`, confidential, standard flow only, no direct grants |
| redirect URI | `<route origin>/api/method/frappe.integrations.oauth2_logins.login_via_keycloak` |
| back-channel logout | not registered — Frappe has no endpoint |
| PKCE | none — Frappe cannot send a challenge |

| Site row field | Source |
| --- | --- |
| `base_url` | `ERPNEXT_SSO_BASE_URL` = `http://keycloak:8080/realms/seamnex` (internal issuer; also the button's truthiness gate) |
| `authorize_url` | `ERPNEXT_SSO_AUTHORIZE_URL` — published IdP origin (**browser** leg) |
| `access_token_url` / `api_endpoint` | `ERPNEXT_SSO_TOKEN_URL` / `_USERINFO_URL` — internal origin (**server** leg) |
| `auth_url_data` | `response_type=code` plus `ERPNEXT_SSO_SCOPE` (`openid profile email`) |
| `user_id_property` | `ERPNEXT_SSO_USER_ID_PROPERTY` (`sub`) |
| `sign_ups` | `ERPNEXT_SSO_SIGN_UPS` (`Allow`) |
| `custom_base_url` | `0` — all endpoints are absolute |

The two origins are split on purpose: the browser must reach the IdP on its published port, the
container on the instance network.

### Provisioning, roles and sessions

- A first sign-in with `Allow` creates an enabled `Website User` with a random password; `Deny`
  answers a 403. An existing disabled user is refused, never re-enabled.
- `Portal Settings.default_role` is the only role a social login confers, and only on creation. A
  `Website User` cannot open `/app`, so an ERP operator still needs `System User` plus roles — an
  in-product action or an `on_social_login` hook. There is no declarative equivalent of EspoCRM's
  `oidcGroupClaim` / `oidcSyncTeams`.
- `ERPNEXT_SESSION_EXPIRY` (default `08:00:00`) is written to `System Settings.session_expiry` and is
  what makes `integration.sessionMaxMinutes: 480` true. The product default is `240:00:00`.
- `ERPNEXT_SSO_DISABLE_LOCAL_LOGIN` (default `false`) keeps the password form, so an IdP outage does
  not lock out the administrator. `sloSupported` is `false`: a sign-out at the platform leaves the
  ERPNext session alive until it expires.

## Sanity check

The host is the instance's route, `<slug>.<SEAMNEX_INGRESS_DOMAIN>`; `erpnext` is the slug both
`integration.apiBaseUrl` and the `ERPNEXT_SITE_NAME` default name.

```powershell
docker ps --filter "label=seamnex.entry=D06-01" --format "table {{.Names}}\t{{.Status}}"
curl.exe -sSk -o NUL -w "%{http_code}`n" https://erpnext.seamnex.localhost:8443/login
docker exec seamnex-erpnext-erpnext-backend-1 bench --site erpnext.seamnex.localhost list-apps
```

Expected: nine containers `Up`, `200` from `/login`, and `frappe` + `erpnext` in `list-apps`. The
platform's `sso_login` check goes further — discovery, the applied client secret, the provider button
on `/login`, then the authorize redirect and an optional token exchange.

## Images

| Image | Digest |
| --- | --- |
| `frappe/erpnext:v16.53.0` | `sha256:b961d69ed3265f44f712f40be58a3e6a22c8662f61a7e1a8097765534e3e9be2` — identical to the catalog `imageDigest`, which is the single authority |
| `mariadb:11.4.13` | `sha256:a8992b5e5404d11e03c54a31d0c1bdea3fd116d283aa6d1a627551f29045309b` — shared with the other PHP/MariaDB entries |
| `redis:7.4.12-alpine` | `sha256:7f7d30b7fdd09467bac58198a1f166dd69789d348d4fe859cccb20e0cf287fbe` — both Redis services |

## Verification notes

Reviewed against `catalog/D06-01.json` and `catalog/env/D06-01.json` on 2026-11-26.

- **Corrected:** the deployment runs **nine** containers, not eight (three Redis/MariaDB sidecars,
  six application containers).
- **Not verifiable from JSON:** `SEAMNEX_VERIFY_TIMEOUT_MS` and the missing per-entry
  `verifyTimeoutMs`; no catalog field exists for a verify timeout.
- **Not verifiable from JSON:** the SSO capability matrix (SAML/PKCE/discovery absent, no
  ID-token signature check) was read from `frappe/frappe` `version-16`, not from a catalog field.
  The catalog records only the resulting `guaranteeClass`/`authMethod` and the `sloSupported: false`
  deviation.
- `D05-01` is named in `sharedWith` but has no record in this repository.
