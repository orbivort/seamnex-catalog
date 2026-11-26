# D01-01 — EspoCRM 10.1.4, vendored upstream tree

| Field | Value |
| --- | --- |
| Product | EspoCRM 10.1.4 (`espocrm/espocrm`) |
| Image | `espocrm/espocrm:10.1.4@sha256:5f8fa9a7…c878c` — pinned by tag **and** digest, referenced by both the web container and the daemon |
| Sidecar | `mariadb:11.4.13@sha256:a8992b5e…5309b` |
| Port | `80` (`httpPort`); the file's `ports: ['8080:80']` is rewritten to `expose: ['80']` |
| Identity | `native` — OIDC, authorization code + PKCE |
| Volumes | `espocrm-data`, `espocrm-custom`, `espocrm-client-custom`, `espocrm-db` |
| Resources | 1 vCPU / 2 GiB / 20 GiB |

`docker-compose.yml` is the reviewed source of truth when the resolver cannot clone
`https://github.com/espocrm/espocrm`. The resolver copies this tree into the per-attempt source
directory and records `SOURCE_REVISION`, so "what was deployed" is provable without the network.

## Deviations from the upstream example

| Concern | Rule |
| --- | --- |
| Pinning | Every image carries a full version; the product image also carries its digest. A floating tag is refused before apply (`unpinned_image`). |
| Host ports | `ports: ['8080:80']` is kept from upstream; the platform rewrites it and asserts zero published ports before apply. |
| Container names | `container_name` is kept verbatim and stripped by the platform, which records the patch. Two instances must not collide. |
| `volumes_from` | Replaced by explicit named volumes — the construct is compose-v1 legacy and is not implemented by the Compose v2+ CLI. |
| Websocket | The `espocrm-websocket` container is omitted: it needs a routable `ws://` endpoint and no host port is published. Core features do not depend on it. |
| Application tree | **Not mounted.** The 10.x image treats a mount on `/var/www/html` as the legacy layout and returns before installing the product or applying `ESPOCRM_CONFIG_*`. Only `data`, `custom` and `client/custom` are mounted. |
| Healthcheck | `bin/command app-check` — the image's own readiness command. `curl /` answers 200-on-302 and would call an uninstalled instance healthy. |

## Environment (`catalog/env/D01-01.json`)

Values are injected by the platform and never live in this tree.

| Variable | Purpose |
| --- | --- |
| `ESPOCRM_SITE_URL` | Published route; also the base of the OIDC redirect URI. Mirrored by the platform into `ESPOCRM_CONFIG_SITE_URL`, which the image applies on every start, so a route change reaches a running instance on restart. Installation-only itself. |
| `ESPOCRM_ADMIN_USERNAME` / `ESPOCRM_ADMIN_PASSWORD` | First administrator. The password is read at install and kept as the break-glass credential while OIDC is active — a production secret, not a demo value. |
| `ESPOCRM_DATABASE_PASSWORD` / `ESPOCRM_DATABASE_ROOT_PASSWORD` | Both required. The root password has no fallback in the file: the upstream `${...:-mariadb_root}` default would give every instance network the same superuser password. |
| `ESPOCRM_DATABASE_PLATFORM` / `_HOST` / `_NAME` / `_USER` | Installer targets, fixed to `Mysql`, `espocrm-db`, `espocrm`, `espocrm`. |
| `ESPOCRM_CONFIG_AUTH_TOKEN_LIFETIME` / `_MAX_IDLE_TIME` | 8 h / 8 h, which is what makes `integration.sessionMaxMinutes: 480` a fact rather than an estimate. The product otherwise has no absolute expiry. |
| `ESPOCRM_CONFIG_AUTHENTICATION_METHOD` | `Oidc` by default; `Espo` restores the local form. |
| `ESPOCRM_CONFIG_OIDC_*` | Client id/secret (`seamnex-<slug>`, platform-generated when empty), the four Keycloak endpoints, claims, team-sync flags and the fallback switches. Only needed while the method is `Oidc`. |

## Updating

Bump `SOURCE_REVISION`, the image tag **and its digest** (both `image:` references) in the same
commit, then deploy. Read the digest from the registry, not the catalog:
`docker buildx imagetools inspect espocrm/espocrm:<version>` prints the index digest that
`imageDigest` records. The render set is content-addressed, so a change appears as a new set rather
than an in-place mutation.

10.1 dropped MariaDB 10.3–10.5 support, so the 11.4 sidecar is the floor for this entry.

## Verification notes

Reviewed against `catalog/D01-01.json` and `catalog/env/D01-01.json` on 2026-11-26.

- **Not in the catalog JSON:** the MariaDB 10.3–10.5 floor and the image's PHP 8.4 / Debian 13 base.
  Both are recorded in `CHANGELOG.md` (2026.11.2), not in a catalog field.
- The OIDC endpoint defaults deliberately split browser and server legs (published IdP origin for
  `AUTHORIZATION_ENDPOINT`, the `keycloak:8080` alias for token/JWKS/userinfo). That split is
  asserted by the schema's `purpose` text, not by a machine-checkable field.
