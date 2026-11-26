# D07-01 — OrangeHRM 5.9, vendored render set

| Field | Value |
| --- | --- |
| Product | OrangeHRM 5.9 (`orangehrm/orangehrm`, release archive from SourceForge) |
| Provenance | root `Dockerfile` at tag `v5.9` (`56e23b3b09e7af29317a0943523b825843fff527`), `php:8.3-apache-bookworm`, archive MD5 `380d85deba0f82174a7db346c9347b87` |
| Image | `seamnex/orangehrm:5.9` — built locally, never pushed, never pulled; upstream publishes no image |
| Sidecar | `mariadb:11.4.13@sha256:a8992b5e…5309b` |
| Port | `80` (`httpPort`), written as `expose: ['80']`; no host port |
| Identity | `fa_gate` — forward-auth gate, product keeps its own login (no OIDC/SAML, no header trust) |
| Volumes | `orangehrm-conf` → `/var/www/html/lib/confs`, `orangehrm-db` → `/var/lib/mysql` |
| Resources | 2 vCPU / 3 GiB / 12 GiB |

The tree exists because a deployment host may have no route to the git server (air-gapped SMB,
filtered corporate network, frozen audit baseline). Upstream publishes neither an image nor a Compose
file, so both are supplied here.

## Deviations from upstream

| Deviation | Why |
| --- | --- |
| `database` service added (`mariadb:11.4.13`, named volume, healthcheck) | Every screen past the login form reads a schema. The installer creates it itself and **refuses** an existing database, so `MARIADB_DATABASE` is deliberately not set. |
| `VOLUME ["/var/www/html"]` removed | It declares the whole application as an anonymous volume, so the installed tree is shadowed by an empty one and every restart presents an uninstalled product. Only `lib/confs` is mounted. |
| First-boot install (`seamnex-bootstrap.sh`, `seamnex-install-once.php`) | Upstream expects a human at the web installer. The install runs once at boot against the `lib/confs/Conf.php` marker (`Config::isInstalled`); a restart never reinstalls, and a failed install still starts Apache so the web installer is reachable with the reason in the log. |
| `HTTPS` derived from `X-Forwarded-Proto` (`seamnex-forwarded-proto.conf`) | The edge terminates TLS and forwards plain HTTP, so `$_SERVER['HTTPS']` was unset and every absolute URL was emitted as `http://`. |
| Application image declared as a local build (`build:` + `pull_policy: never`) | The artefact must come from this tree during the deployment. The tag is in the platform's namespace (`seamnex/…`), because a reference under the product's name can only resolve to a third party's repackaging — which is what once ran here. |
| Licence evidence in the image (`IMAGE-LICENCES.md`, OCI labels) | The label states the platform's `GPL-3.0-only` and names the product's licence as an unverified reference. |

Notes for editors: the install runs as root (it writes `installer/cli_install_config.yaml`) and hands
`lib/confs`, `src/cache`, `src/log` and `src/config` back to `www-data`. The deprecated
`installer/cli_install.php` is used because its replacement (`installer/console install:on-new-database`)
refuses to run without a TTY. **Nothing here patches the product** — a modified PHP file would make
the image a modified version under GPL-3.0 §5 and break the `md5sum -c` "unmodified archive" claim.

## Build and pinning

The `orangehrm` service declares `build:`, so the platform builds this tree during every deployment,
after admission and before anything starts. The build is issued against BuildKit, never the socket
proxy (which answers 403; ADR-BE-40 §7); `--load` returns the image to the host daemon, which is what
lets `apply` start it.

| Stage | Proof |
| --- | --- |
| `build` | `docker compose build` ran from the resolved tree; the produced id is recorded in the job log |
| `apply`, before `up` | the tag still resolves to that id — a retag or substitution is refused with `IMAGE_DIGEST_MISMATCH` |
| `apply`, after `up` | the containers are running the artefact this deployment built |
| `record` | `instance_service.image_digest` holds the id that later deploys compare against |

The first deploy is trust-on-first-use. Later deploys report a changed id as drift rather than
refusing it, because a rebuild without the layer cache legitimately produces a new id; a tag that no
longer resolves to what this run built is refused. `imageDigest` in `catalog/D07-01.json` is therefore
reference data — the enforced baseline is a property of the host that built the artefact.

An image that lives on one host only means every host that may run it must build its own copy (a
second node needs the deployment or "Prefetch images"), and an appliance export must carry it
(`docker save seamnex/orangehrm:5.9`).

## Environment (`catalog/env/D07-01.json`)

| Variable | Purpose |
| --- | --- |
| `ORANGEHRM_DATABASE_ROOT_PASSWORD` | Required; the installer has no fallback. The privileged account creates the schema and the application user. |
| `ORANGEHRM_DATABASE_PASSWORD` | Password of the `orangehrm` application user; the application never runs as root. |
| `ORANGEHRM_DATABASE_HOST` / `_PORT` / `_NAME` / `_USER` | Target database; default `orangehrm-db:3306`, schema `orangehrm`, user `orangehrm`. `_NAME` must name an empty schema — the installer refuses one that already exists. |
| `ORANGEHRM_ADMIN_USERNAME` / `_PASSWORD` / `_EMAIL` / `_FIRST_NAME` / `_LAST_NAME` | First administrator and its employee record. The e-mail must be monitored; it also receives product notifications. |
| `ORANGEHRM_ORGANIZATION_NAME` / `_COUNTRY` | Organisation written at install. The country is an ISO 3166-1 alpha-2 code (`DE`, not `Germany`). |
| `ORANGEHRM_DATA_ENCRYPTION` | Default `false`. When on, the key file lives in `lib/confs/cryptokeys`, so the config volume becomes mandatory in every backup. |
| `ORANGEHRM_REGISTRATION_CONSENT` | Default `false` — the platform does not phone home on the operator's behalf (upstream defaults to yes). |
| `ORANGEHRM_INSTALLER_LICENSE_AGREED` | Declared `required` with default `"true"`. The installer proceeds only when the value is literally `true`; otherwise it logs why and leaves the web installer reachable. |

## Open licence actions

A licence review cannot pass until these close — they are properties of the artefact, not of the code
that deploys it:

1. **Read the archive's licence:** `docker run --rm --entrypoint cat seamnex/orangehrm:5.9 /var/www/html/LICENSE`
   (or `license/`). The catalog says `GPL-3.0`; nothing in this tree proves it. `GPL-2.0-only` would
   make this image non-conveyable under a `GPL-3.0-only` platform.
2. **Inventory the bundled components** from `composer.json` and `vendor/composer/installed.json`, and
   add a row per obligation-carrying licence to `IMAGE-LICENCES.md`.
3. **Record the licence review** in the instance's register (`license_record`).

## Updating

Bump `SOURCE_REVISION`, the `OHRM_VERSION` / `OHRM_MD5` build arguments, the image tag and `imageRef`
in `catalog/D07-01.json` in the same commit, then deploy: the `build` stage rebuilds and re-tags and
the next `record` writes the new id. Nothing is published, so there is no digest to push.

## Verification notes

Reviewed against `catalog/D07-01.json` and `catalog/env/D07-01.json` on 2026-11-26.

- **Inconsistency to resolve:** the env schema declares `ORANGEHRM_INSTALLER_LICENSE_AGREED` as
  `required` with default `"true"`, while the installer treats any other value as "no consent
  recorded". As declared, a fresh deployment therefore installs without an operator action, which
  contradicts the intent stated in that variable's own `purpose` text (a script must not tick a
  licence acceptance). This README can only report the two states; they do not agree.
- **Not in the catalog JSON:** the commit hash, archive MD5, `php:8.3-apache-bookworm` base and the
  SourceForge URL shape. They come from `Dockerfile` and `SOURCE_REVISION`.
- `imageDigest` is reference data for this local build, per the record's own `notes`.
