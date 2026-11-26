# D02-01 — Mautic 7.2.2, vendored render set

| Field | Value |
| --- | --- |
| Product | Mautic 7.2.2 (`mautic/mautic`) |
| Image | `seamnex/mautic:7.2.2` — built locally from this tree over `mautic/mautic:7.2.2-apache` |
| Sidecar | `mariadb:11.4.13@sha256:a8992b5e…5309b` |
| Port | `80` (`httpPort`), written as `expose: ['80']`; no host port |
| Identity | `fa_gate` — forward-auth gate, product keeps its own login |
| Volumes | `mautic-config`, `mautic-media`, `mautic-logs`, `mautic-db` |
| Resources | 2 vCPU / 2 GiB / 20 GiB |

`docker-compose.yml` is the reviewed source of truth when the resolver cannot clone
`https://github.com/mautic/mautic`. The resolver copies this tree into the per-attempt source
directory and records `SOURCE_REVISION`, so "what was deployed" is provable without the network.

## Deviations from upstream

| Concern | Rule |
| --- | --- |
| Build | The application is a thin layer over the pinned base (`Dockerfile`): `mautic-web` declares `build:` + `pull_policy: never`, so the executor builds `seamnex/mautic:7.2.2` in the deployment's `build` stage and refuses any other image under that tag (ADR-BE-40). The build goes through `SEAMNEX_BUILDKIT_ADDR`, never the socket proxy (which answers 403). "Prefetch images" builds it ahead of the window. |
| Host ports | No `ports:` at all; `expose: ['80']` is what `httpPort` refers to. |
| Database | Added here. Mautic's installer creates tables but neither the schema nor the account, so the MariaDB container creates both and a healthcheck gates the application on `--innodb_initialized`. |
| First boot | `seamnex-bootstrap.sh` runs `bin/console mautic:install` once, then delegates to the image's `/entrypoint.sh`. The image ships the product uninstalled and only starts Apache. |
| Marker | `/var/www/html/config/local.php` with `db_driver` and `site_url`; a restart never reinstalls. |
| Proxy trust | `SEAMNEX_MAUTIC_TRUSTED_PROXIES` (default: the three private ranges) is written into `trusted_proxies` on every start. Without it the post-login redirect is `http://<route>:<edge port>`, which nothing serves. The name cannot be `MAUTIC_*`: those map to configuration parameters and a comma list breaks every request. |
| API | `SEAMNEX_MAUTIC_API_ENABLED` (default `1`) writes `api_enabled` and `api_enable_basic_auth` into `local.php`, which the per-user provisioning path needs. |
| Cron | A second container runs `DOCKER_MAUTIC_ROLE=mautic_cron`; campaigns and segments are sent from scheduled commands. The wrapper bounds the image's unbounded wait for an installation, so a failed install surfaces as a restart loop. |
| Volumes | `/var/www/html/var` as a whole is **not** mounted: `var/cache/prod` ships pre-built in the image and an empty volume would shadow it. |
| Licences | `IMAGE-LICENCES.md` travels inside the image and is named by the OCI `licenses` label. The product's own terms in the base image are unread, so the label names them `LicenseRef-Mautic-Upstream` (`THIRD-PARTY-NOTICES.md` §4.2, §6.2). |

## Environment (`catalog/env/D02-01.json`)

| Variable | Purpose |
| --- | --- |
| `MAUTIC_SITE_URL` | Published route including the edge port. Mautic builds every absolute URL in an e-mail from it. Installation-only. |
| `MAUTIC_DB_HOST` / `_PORT` / `_NAME` / `_USER` | Database target; must agree with the sidecar (`mautic-db:3306`, schema `mautic`, user `mautic`). |
| `MAUTIC_DB_PASSWORD` | Application account; injected into both containers so they cannot disagree. |
| `MAUTIC_DATABASE_ROOT_PASSWORD` | Required; the sidecar has no fallback. Must differ from `MAUTIC_DB_PASSWORD`. |
| `MAUTIC_DB_TABLE_PREFIX` | Locked to **empty**. The 7.2.2 installer records the prefix in `local.php` but creates the schema unprefixed; `mautic_` produced a schema-less install whose next migration failed on `mautic.mautic_email_stats`. |
| `MAUTIC_ADMIN_USERNAME` / `_PASSWORD` / `_EMAIL` / `_FIRSTNAME` / `_LASTNAME` | First administrator, read once by the install. The e-mail must be monitored: Mautic writes failed sends and cron errors to it. |

## Recovering a failed installation

The installer refuses a schema whose tables already exist, so repair a half-finished install by
removing the database volume and deploying again:

```bash
docker volume rm seamnex-mautic_mautic-db
```

## Updating

Bump the base tag in `Dockerfile` and the `image:` tags in `docker-compose.yml`, refresh
`SOURCE_REVISION`, then update `imageRef` in `catalog/D02-01.json` and `appliesTo.imageRef` in
`catalog/env/D02-01.json` in the same commit. The next deployment or prefetch builds the new tag;
nothing is pulled and nothing is pushed.

## Verification notes

Reviewed against `catalog/D02-01.json` and `catalog/env/D02-01.json` on 2026-11-26.

- **Undeclared variables:** `SEAMNEX_MAUTIC_TRUSTED_PROXIES` and `SEAMNEX_MAUTIC_API_ENABLED` are read
  by `docker-compose.yml` but are **not** declared in `catalog/env/D02-01.json`, so only the file's
  defaults apply and a drift scan would report them as undeclared.
- `imageDigest` is reference data for this local build (the record's own `notes` say so); the enforced
  pin is the image id the host built.
- The volume-removal example assumes a project name of `seamnex-<slug>`; the compose file itself
  declares `name: mautic`, which the renderer overrides.
