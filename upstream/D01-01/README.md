# D01-01 — EspoCRM, vendored upstream tree

`docker-compose.yml` in this directory is the reviewed source of truth for a D01-01 deployment when
the resolver cannot clone `https://github.com/espocrm/espocrm` (no route to the git host, or an
air-gapped host). The resolver copies this tree into the per-attempt source directory and records
`SOURCE_REVISION` as the source revision, so "what was deployed" is provable without the network.

| Concern | Rule |
| --- | --- |
| Images | Every image is pinned to a full version **and** a digest: the product image `espocrm/espocrm:10.1.4@sha256:5f8fa9a7…` (it is referenced twice — the web container and the daemon — so both must move together) and the MariaDB sidecar `mariadb:11.4.13@sha256:a8992b5e…`. The compose policy refuses a floating tag (`unpinned_image`) before anything is applied; the digest additionally survives a tag being repointed at another artefact. |
| Host ports | `ports: ['8080:80']` is kept from upstream; the platform rewrites it to `expose: ['80']` and asserts zero published ports immediately before apply. |
| Container names | `container_name` is kept from upstream and stripped by the platform, which records the patch. Two instances of one product must not collide. |
| `volumes_from` | Replaced by explicit named volumes: the construct is compose-v1 legacy and the Compose v2+ CLI does not implement it. |
| Websocket | The `espocrm-websocket` container is omitted. It needs a routable `ws://` endpoint and the platform publishes no host ports; EspoCRM's core features do not depend on it. |
| Application tree | **Not mounted.** The 10.x image treats a mount on `/var/www/html` as the legacy layout: its entrypoint returns before installing the product and before applying `ESPOCRM_CONFIG_*`. Only `data`, `custom` and `client/custom` are mounted, so the install runs and the environment reaches the product. |
| Healthcheck | `bin/command app-check`, the image's own readiness command: `curl /` answers 200-on-302 and calls an uninstalled instance healthy. |

## What the operator must supply

The catalog entry `D01-01` declares the variables this file reads (`env_schema_var`), so the values are
injected by the platform and never live in this tree:

- `ESPOCRM_SITE_URL` — the published route, e.g. `http://espocrm.<ingress domain>:<edge port>`.
- `ESPOCRM_ADMIN_USERNAME` / `ESPOCRM_ADMIN_PASSWORD` — used only on the first install.
- `ESPOCRM_DATABASE_PASSWORD` — secret; reconciles the MariaDB user with the application.
- `ESPOCRM_DATABASE_ROOT_PASSWORD` — secret and **required** by the catalog entry: the file's
  `${...:-mariadb_root}` fallback is a well-known value, so leaving it unset would give every
  database in the instance network the same superuser password.

To update the product, bump `SOURCE_REVISION`, the image tag **and its digest** in the same commit
(both `image:` references), then deploy again; the render set is content-addressed, so the change is
visible as a new set rather than an in-place mutation of the previous one. Read the digest from the
registry rather than the catalog: `docker buildx imagetools inspect espocrm/espocrm:<version>` prints
the index digest, which is the value `imageDigest` records.

10.1 cannot run against an older database: it drops MariaDB 10.3–10.5 support, so the 11.4 sidecar
above is the floor for this entry. The image itself is PHP 8.4 on Debian 13 (trixie) — the base
10.0.11 already used, so the bump moves no third-party layer.
