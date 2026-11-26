# Third-Party Notices

| | |
| --- | --- |
| Document version | 1.1 |
| Last reviewed | 2026-11-26 |
| Applies to | this repository at its current commit (catalog schema **D01-01 … D11-01**) |
| Owner | platform engineering — update this file in the same commit as the change it describes |
| Status of this document | inventory of fact, **not legal advice** |
| Licence of this document | GPL-3.0-only, like the rest of this repository (`LICENSE`) |

This file records every third-party component this repository selects, vendors, builds or
pulls at deployment time, together with the licence terms that travel with it. It is written to be
read by two audiences: an operator who has to answer "what am I running, and under what terms", and
an engineer who has to keep the answer true after the next version bump.

---

## 1. Scope, sources and conventions

### 1.1 What this repository actually contains

This repository ships **no application code of its own**. It contains:

| Directory | Contents |
| --- | --- |
| `catalog/*.json` | one record per product: version, image reference, digest, licence identifier, provenance |
| `catalog/env/*.json` | the environment declarations those records reference |
| `upstream/<id>/` | vendored Compose definitions, and for two entries a `Dockerfile` that wraps an upstream image |
| `LICENSE` | the verbatim GNU GPL v3 text that licenses this repository |

There is no `package.json`, `composer.json`, `Gemfile`, `requirements.txt` or lockfile in the tree.
Consequently there is **no repository-level source dependency graph** to inventory: every third-party
component here is either (a) a product selected by a catalog record, or (b) a container image named in
a vendored Compose file or `Dockerfile`. If a language manifest is added later, its resolved inventory
must be added as a new section in the same pull request, with a row in §2.1.

### 1.2 Where each fact comes from

Each entry below is anchored to a file, so a reviewer can check it rather than trust it:

| Field | Authority |
| --- | --- |
| Product, version, image reference | `catalog/<id>.json` (`product`, `upstream.currentVersion`, `imageRef`) |
| Image digest | `catalog/<id>.json` (`imageDigest`) — the reviewed artefact for the product image. The catalog record has no sidecar slot: a sidecar's digest lives in the Compose `image:` reference (§5) |
| Licence identifier | `catalog/<id>.json` (`licenseId`) — the short SPDX form, normalised here to a full SPDX expression |
| Services and sidecar images | `upstream/<id>/docker-compose.yml` (tag **and digest** in the `image:` reference) |
| Base image of a locally built image | `upstream/<id>/Dockerfile` |
| Deeper image contents | the image itself; see §6.9 for how to enumerate them |

### 1.3 Evidence status — read this column before quoting a licence

| Status | Meaning |
| --- | --- |
| `repo` | the identifier was read from the upstream project's licence file and is mirrored in `catalog/<id>.json` |
| `inherited` | the terms belong to a base image or OS layer that this repository does not modify, and are passed through unchanged |
| `unverified` | the identifier is recorded, but **nobody has read the licence inside the artefact that is actually deployed**; do not rely on it for redistribution |
| `tag-dependent` | the licence depends on which patch release the tag resolved to; the reference must be pinned before the answer is fixed |

`unverified` rows are not a formality. `upstream/D02-01/IMAGE-LICENCES.md` and
`upstream/D07-01/IMAGE-LICENCES.md` both state, in the image's own documentation, that the product's
licence was never read out of the built artefact, and why that matters (§7.2).

### 1.4 Out of scope

The edge proxy, the identity provider, the control plane and the deployment host are **not** in this
repository and are therefore not listed here. Components that a product reaches over the network
(a customer's SMTP relay, an identity provider, a database the operator supplies) are also out of
scope: this file covers what this repository puts on the host.

---

## 2. Inventory at a glance

### 2.1 Direct components

| # | Component | Version | Licence (SPDX) | Role | Used by | Evidence |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | EspoCRM | 10.1.4 | `AGPL-3.0-only` | CRM application | D01-01 | `repo` |
| 2 | Mautic | 7.2.2 (image `7.2.2-apache`) | `GPL-3.0-only` | marketing automation | D02-01 | `unverified` |
| 3 | ERPNext | 16.53.0 | `GPL-3.0-only` | ERP application | D06-01 | `repo` |
| 4 | Frappe Framework | 16.53.0 (bundled) | `GPL-3.0-only` | ERPNext platform | D06-01 | `repo` |
| 5 | OrangeHRM | 5.9 | `GPL-3.0-only` | HR application | D07-01 | `unverified` |
| 6 | OpenProject | 17.9.6 | `GPL-3.0-only` | project management | D09-01 | `repo` |
| 7 | Metabase | 0.65.2 | `AGPL-3.0-only` | BI / analytics | D11-01 | `repo` |
| 8 | MariaDB | 11.4.13 | `GPL-2.0-only` (server), `LGPL-2.1-only` (client libs) | database sidecar | D01-01, D02-01, D06-01, D07-01 | `inherited` |
| 9 | Redis | 7.4.12 (`7.4.12-alpine`) | `RSALv2` **or** `SSPL-1.0` for 7.4+; `BSD-3-Clause` for ≤ 7.2.4 | cache and job queue | D06-01 | `inherited` |
| 10 | PostgreSQL | 18.7 (`18.7-alpine`) | `PostgreSQL` | application database | D11-01 | `inherited` |
| 11 | PHP | 8.3 | `PHP-3.01` | runtime of the built images | D02-01, D07-01 | `inherited` |
| 12 | Apache HTTP Server | 2.4 | `Apache-2.0` | web server of the built images | D02-01, D07-01 | `inherited` |
| 13 | Debian GNU/Linux | 12 (bookworm) | mixed (`GPL-2.0-only`, `LGPL-2.1-only`, `MIT`, `BSD-3-Clause`, …) | base OS of the built images | D02-01, D07-01 | `inherited` |
| 14 | Alpine Linux | 3.x (`-alpine` tags) | mixed (`MIT` for musl, `GPL-2.0-only` for BusyBox) | base OS of the sidecar images | D06-01, D11-01 | `inherited` |

Rows `1`–`7` are the products the catalog offers. Rows `8`–`14` are infrastructure this repository
declares; they are dependencies of the catalog entries, not of the products.

### 2.2 Known verification gaps

These are tracked deliberately rather than smoothed over. Each one is a precondition for
redistributing the affected image.

| ID | Gap | Consequence | Owner action |
| --- | --- | --- | --- |
| G1 | Mautic 7.2.2: the `LICENSE` inside `mautic/mautic:7.2.2-apache` was never read (`upstream/D02-01/IMAGE-LICENCES.md`) | if the product is `GPL-2.0-only`, the built layer stack is not conveyable (§7.2) | read `LICENSE` and `vendor/composer/installed.json` in the image, replace `LicenseRef-Mautic-Upstream` in the OCI label, record the review here |
| G2 | OrangeHRM 5.9: the archive's `LICENSE` was never read (`upstream/D07-01/IMAGE-LICENCES.md`) | same as G1 | read `license/` and `composer.json` in the built image, replace `LicenseRef-OrangeHRM-Upstream` |
| G3 | Redis is pinned by tag **and digest** in the Compose file, so family and artefact are both fixed at `RSALv2`/`SSPL-1.0` — but `catalog/D06-01.json` records no sidecar digest, and the licence is not OSI-approved (§7.5) | a reviewer reading the catalog alone cannot see which Redis artefact runs | add a sidecar digest to the catalog record, or re-pin `7.2.x` (`BSD-3-Clause`) / `8.x` (`AGPL-3.0-only`) if a different licence baseline is required |
| G4 | Metabase: the AGPL text applies to the open-source edition; the repository's `enterprise/` tree is under the Metabase Commercial Licence | an operator who enables a paid feature changes the terms of the instance | confirm the edition at deployment; enterprise features are not enabled by this repository |
| G5 | SQLite `sample-database.sqlite` ships in the Metabase image and is extracted even with `MB_LOAD_SAMPLE_CONTENT=false` (`upstream/D11-01/README.md`). SQLite is `blessing` (public domain) | none, but it is an uninventoried component in the image | note it here; no attribution obligation |
| G6 | Ruby gems, Python packages and Debian/Alpine package sets inside the upstream images are not enumerated | a licence with an obligation (LGPL, source-available) could hide there | generate an SBOM per image (§6.9) and add rows |

---

## 3. Dependency map (direct → transitive)

The arrows read "is built from / runs on". Indentation shows what is inside the image.

```text
D01-01 EspoCRM
├── espocrm/espocrm:10.1.4          EspoCRM 10.1.4 + PHP + Apache + Debian   [inherited]
└── mariadb:11.4.13                  MariaDB 11.4.13 + Debian                  [inherited]

D02-01 Mautic
├── seamnex/mautic:7.2.2 (built here)
│   └── mautic/mautic:7.2.2-apache   Mautic 7.2.2 + PHP + Apache + Debian      [inherited, unverified]
└── mariadb:11.4.13                  MariaDB 11.4.13 + Debian                  [inherited]

D06-01 ERPNext
├── frappe/erpnext:v16.53.0          ERPNext + Frappe + Python + nginx + Node + Debian  [inherited]
├── mariadb:11.4.13                  MariaDB 11.4.13 + Debian                  [inherited]
└── redis:7.4.12-alpine              Redis 7.4.12 + Alpine                     [inherited]

D07-01 OrangeHRM
├── seamnex/orangehrm:5.9 (built here)
│   └── php:8.3-apache-bookworm      PHP 8.3 + Apache 2.4 + Debian 12          [inherited]
│       └── OrangeHRM 5.9            release archive from SourceForge           [unverified]
└── mariadb:11.4.13                  MariaDB 11.4.13 + Debian                  [inherited]

D09-01 OpenProject
└── openproject/openproject:17.9.6   OpenProject + Rails/Puma + PostgreSQL 17
                                     + memcached + Apache + Ruby gems + Debian  [inherited]

D11-01 Metabase
├── metabase/metabase:v0.65.2        Metabase + JVM + Jetty + JDBC drivers     [inherited]
└── postgres:18.7-alpine             PostgreSQL 18.7 + Alpine                  [inherited]
```

---

## 4. Catalog applications

### 4.1 EspoCRM — `D01-01`

| Field | Value |
| --- | --- |
| Component | EspoCRM |
| Version | 10.1.4 |
| Image | `espocrm/espocrm:10.1.4@sha256:5f8fa9a708893b9fbfe55a778f24c952a6a73b94c97fb126eccd21aa936c878c` — pinned by tag **and digest** in the Compose file (both the web container and the daemon) |
| Licence | `AGPL-3.0-only` (catalog records the deprecated short form `AGPL-3.0`) |
| Copyright | EspoCRM, Inc. and contributors |
| Source | <https://github.com/espocrm/espocrm> · image build: <https://github.com/espocrm/espocrm-docker> · releases: <https://github.com/espocrm/espocrm/releases> |
| Declared in | `catalog/D01-01.json`, `upstream/D01-01/docker-compose.yml` |
| Evidence | `repo` — upstream `LICENSE`, mirrored as `licenseId` |
| Notice | Reproduce the upstream `LICENSE` in full with any redistribution. §13 (network interaction) applies to a **modified** version offered as a service; this repository vendors the Compose definition only and does not modify the product. |

### 4.2 Mautic — `D02-01`

| Field | Value |
| --- | --- |
| Component | Mautic |
| Version | 7.2.2, base image tag `7.2.2-apache` |
| Image | `seamnex/mautic:7.2.2`, digest `sha256:0cc3f586d03ee5fa3bd1d0e3b2547d26e4dd573a1fd15aae87075bf604529a7e`; built from `mautic/mautic:7.2.2-apache` |
| Licence | `GPL-3.0-only` as recorded in the catalog — **not confirmed inside the image** |
| Copyright | Mautic Contributors |
| Source | <https://github.com/mautic/mautic> · image build: <https://github.com/mautic/docker-mautic> · releases: <https://github.com/mautic/mautic/releases> |
| Declared in | `catalog/D02-01.json`, `upstream/D02-01/Dockerfile`, `upstream/D02-01/docker-compose.yml` |
| Evidence | `unverified` — see G1 and `upstream/D02-01/IMAGE-LICENCES.md` |
| Notice | The OCI label carries `GPL-3.0-only AND LicenseRef-Mautic-Upstream`; the custom reference is a placeholder for a licence nobody has read. Resolve G1 before redistribution. Conveying the image requires the Corresponding Source or a written offer (GPL-3.0 §6). |
| Platform additions | `seamnex-bootstrap.sh` (GPL-3.0-only, this repository) — added, not a patch of the product |

### 4.3 ERPNext and the Frappe Framework — `D06-01`

| Field | Value |
| --- | --- |
| Component | ERPNext |
| Version | 16.53.0 |
| Image | `frappe/erpnext:v16.53.0`, digest `sha256:b961d69ed3265f44f712f40be58a3e6a22c8662f61a7e1a8097765534e3e9be2` |
| Licence | `GPL-3.0-only` |
| Copyright | Frappe Technologies Pvt. Ltd. and contributors |
| Source | <https://github.com/frappe/erpnext> · image build: <https://github.com/frappe/frappe_docker> · releases: <https://github.com/frappe/erpnext/releases> |
| Declared in | `catalog/D06-01.json`, `upstream/D06-01/docker-compose.yml` |
| Evidence | `repo` |
| Notice | Reproduce the upstream `LICENSE`. The image is conveyed with the platform's first-boot script inline in the Compose command; that script is GPL-3.0-only (this repository) and is not a modification of ERPNext. |

| Field | Value |
| --- | --- |
| Component | Frappe Framework (transitive) |
| Version | 16.53.0 — the branch the pinned image is built from |
| Licence | `GPL-3.0-only` |
| Copyright | Frappe Technologies Pvt. Ltd. and contributors |
| Source | <https://github.com/frappe/frappe> |
| Why it is listed | `bench new-site --install-app erpnext` installs onto Frappe; the site is unusable without it, so it is a runtime dependency even though this repository never names it as a service |
| Notice | Same obligations as ERPNext; it is inside the same image |

### 4.4 OrangeHRM — `D07-01`

| Field | Value |
| --- | --- |
| Component | OrangeHRM |
| Version | 5.9 (upstream commit `56e23b3b09e7af29317a0943523b825843fff527`) |
| Image | `seamnex/orangehrm:5.9`, digest `sha256:bd76663f62bc203770dee8170ede981045ee8e7d782f228d13c87e40e8527a75` |
| Artefact | `orangehrm-5.9.zip` from SourceForge, MD5 `380d85deba0f82174a7db346c9347b87`, verified at build time |
| Licence | `GPL-3.0-only` as recorded in the catalog — **the archive's own `LICENSE` was never read** |
| Copyright | OrangeHRM, Inc. |
| Source | <https://github.com/orangehrm/orangehrm> · archive: <https://sourceforge.net/projects/orangehrm/files/stable/5.9/> · releases: <https://github.com/orangehrm/orangehrm/releases> |
| Declared in | `catalog/D07-01.json`, `upstream/D07-01/Dockerfile`, `upstream/D07-01/docker-compose.yml` |
| Evidence | `unverified` — see G2 and `upstream/D07-01/IMAGE-LICENCES.md` |
| Notice | The OCI label carries `GPL-3.0-only AND LicenseRef-OrangeHRM-Upstream`. Resolve G2 before redistribution. Nothing in the repository reproduces the product's licence text, so it must be obtained from the archive. |
| Platform additions | `seamnex-bootstrap.sh`, `seamnex-install-once.php`, the Apache `X-Forwarded-Proto` rule (GPL-3.0-only, this repository) — added, not patches |

### 4.5 OpenProject — `D09-01`

| Field | Value |
| --- | --- |
| Component | OpenProject |
| Version | 17.9.6 |
| Image | `openproject/openproject:17.9.6@sha256:c4802aca392506c389ff10a2766caf949000383f20ac752ee5913e1793fdff98` — the first definition to pin the digest in the Compose file; the MariaDB, Redis and PostgreSQL sidecars follow the same rule (§5) |
| Licence | `GPL-3.0-only` |
| Copyright | OpenProject GmbH |
| Source | <https://github.com/opf/openproject> · image: <https://hub.docker.com/r/openproject/openproject> · releases: <https://github.com/opf/openproject/releases> |
| Declared in | `catalog/D09-01.json`, `upstream/D09-01/docker-compose.yml` |
| Evidence | `repo` |
| Notice | Reproduce the upstream `LICENSE` when conveying the image. Editions: the Community edition is GPL-3.0; Enterprise add-ons are separately licensed and are not enabled by this repository. |
| Bundled (transitive) | Rails/Puma, PostgreSQL 17, memcached, Apache, supervisord, Ruby gems — §6.5 |

### 4.6 Metabase — `D11-01`

| Field | Value |
| --- | --- |
| Component | Metabase |
| Version | 0.65.2 |
| Image | `metabase/metabase:v0.65.2`, digest `sha256:b62ac14872e9992d28218a2264d99484e0707064aae9fb594d8fbb3bc1518a9e` |
| Licence | `AGPL-3.0-only` for the open-source edition; `enterprise/` in the same repository is under the Metabase Commercial Software Licence |
| Copyright | Metabase, Inc. |
| Source | <https://github.com/metabase/metabase> · releases: <https://github.com/metabase/metabase/releases> |
| Declared in | `catalog/D11-01.json`, `upstream/D11-01/docker-compose.yml` |
| Evidence | `repo` |
| Notice | Reproduce the upstream `LICENSE` (AGPL-3.0) with any redistribution. §13 applies if an operator modifies the application and offers it as a network service; the platform does not modify it. See G4 and G5. |
| Bundled (transitive) | JVM, Jetty, Clojure runtime, JDBC drivers, `sample-database.sqlite` — §6.6 |

---

## 5. Database, cache and runtime sidecars

### 5.1 MariaDB 11.4

| Field | Value |
| --- | --- |
| Component | MariaDB Server (and the client libraries in the same image) |
| Version | 11.4.13 |
| Image | `mariadb:11.4.13@sha256:a8992b5e5404d11e03c54a31d0c1bdea3fd116d283aa6d1a627551f29045309b` — pinned by tag **and digest** in the Compose file of D01-01, D02-01, D06-01 and D07-01. The catalog record has no sidecar digest field, so those four files are the authority for this artefact. |
| Licence | Server `GPL-2.0-only`; client libraries `LGPL-2.1-only`; the image also carries Debian packages |
| Copyright | MariaDB Foundation / MariaDB plc and contributors; contains code originally copyright MySQL AB and Monty Program Ab |
| Source | <https://github.com/MariaDB/server> · image build: <https://github.com/MariaDB/mariadb-docker> · <https://hub.docker.com/_/mariadb> |
| Used by | D01-01, D02-01, D06-01, D07-01 — always as a **separate container** reached over TCP |
| Evidence | `inherited` |
| Notice | Redistribution of the image requires the GPL-2.0 text and the corresponding source or a written offer. The `GPL-2.0-only` identifier is why this component must never be linked into platform code — see §7.2. |

### 5.2 Redis 7 (cache and queue)

| Field | Value |
| --- | --- |
| Component | Redis |
| Version | 7.4.12 |
| Image | `redis:7.4.12-alpine@sha256:7f7d30b7fdd09467bac58198a1f166dd69789d348d4fe859cccb20e0cf287fbe` — pinned by tag **and digest** in `upstream/D06-01/docker-compose.yml` (both Redis services) |
| Licence | `RSALv2` **or** `SSPL-1.0` (dual) from 7.4 onward; `BSD-3-Clause` up to and including 7.2.4 |
| Copyright | Redis Ltd.; earlier releases copyright Salvatore Sanfilippo and contributors |
| Source | <https://github.com/redis/redis> · <https://hub.docker.com/_/redis> · licence history: <https://redis.io/legal/licenses/> |
| Used by | D06-01 (`erpnext-redis-cache`, `erpnext-redis-queue`) |
| Evidence | `inherited` — the tag is pinned by digest, so the family is no longer resolution-dependent; the licence *policy* question stays open in G3 |
| Notice | Neither `RSALv2` nor `SSPL-1.0` is an OSI-approved licence. Internal use as a cache and queue, with no modification and no offering of Redis itself as a service, is within the grant; do not expose Redis as a product. If a permissive or OSI baseline is required, pin `7.2.x` (BSD-3-Clause) or move to Redis 8 (`AGPL-3.0-only`) as a reviewed change. |

### 5.3 PostgreSQL 18

| Field | Value |
| --- | --- |
| Component | PostgreSQL |
| Version | 18.7 (per `upstream/D11-01/README.md`) |
| Image | `postgres:18.7-alpine@sha256:0115267ab5a509022df9319c77be365cef335adb71556c75f60f5360ee247e71` — pinned by tag **and digest** in `upstream/D11-01/docker-compose.yml` |
| Licence | `PostgreSQL` (permissive, BSD-style) |
| Copyright | The PostgreSQL Global Development Group; portions The Regents of the University of California |
| Source | <https://github.com/postgres/postgres> · image build: <https://github.com/docker-library/postgres> · <https://hub.docker.com/_/postgres> |
| Used by | D11-01 (`db` sidecar) |
| Evidence | `inherited` |
| Notice | The copyright notice and the permission paragraphs must appear in all copies. Exact text in §8.2. The image's Alpine base is covered in §6.7. |

---

## 6. Base images, OS layers and build-time packages

These entries are `inherited`: this repository selects the base and adds files, but does not modify
or relicense what is underneath. They are listed because a recipient of `docker save` receives them.

### 6.1 `php:8.3-apache-bookworm` — base of the OrangeHRM image

| Component | Version | Licence | Copyright | Source |
| --- | --- | --- | --- | --- |
| PHP | 8.3 | `PHP-3.01` | The PHP Group | <https://github.com/php/php-src>, <https://hub.docker.com/_/php> |
| Apache HTTP Server | 2.4 | `Apache-2.0` | The Apache Software Foundation | <https://httpd.apache.org/>, <https://github.com/apache/httpd> |
| Debian GNU/Linux | 12 (bookworm) | mixed per package | Debian Project and package authors | <https://www.debian.org/>, <https://github.com/debuerreotype/docker-debian-artifacts> |

### 6.2 `mautic/mautic:7.2.2-apache` — base of the Mautic image

The image bundles Mautic 7.2.2 (§4.2) on a PHP/Apache/Debian stack equivalent to §6.1. The
project's own licence inventory for that stack is not reproduced upstream; inherit §6.1 for the
runtime layers, and treat the Mautic layer as `unverified` until G1 is closed.

### 6.3 `espocrm/espocrm:10.1.4`

EspoCRM 10.1.4 (§4.1) on `php:8.4-apache-trixie` — PHP 8.4.28 and Debian 13 (trixie), read out of the
image itself. The previous pin, 10.0.11, was on the same base (PHP 8.4.27 / trixie), so the bump moves
no third-party layer. EspoCRM's own image also carries Composer dependencies under their respective
licences; they are not inventoried in this repository.

### 6.4 `frappe/erpnext:v16.53.0`

Built by `frappe/frappe_docker`. Beyond ERPNext and Frappe (§4.3) it contains Python, nginx, Node.js
(for socket.io), a MariaDB client library, and a Debian package set. Each is under its own licence;
none is enumerated here (G6).

### 6.5 `openproject/openproject:17.9.6`

A single-container stack under supervisord (`upstream/D09-01/README.md`):

| Bundled component | Licence (SPDX) | Copyright | Source |
| --- | --- | --- | --- |
| PostgreSQL 17 | `PostgreSQL` | The PostgreSQL Global Development Group | <https://www.postgresql.org/> |
| memcached | `BSD-3-Clause` | Danga Interactive and contributors | <https://memcached.org/> |
| Apache HTTP Server | `Apache-2.0` | The Apache Software Foundation | <https://httpd.apache.org/> |
| Ruby on Rails / Puma | `MIT` | Rails Core Team; Puma contributors | <https://rubyonrails.org/>, <https://puma.io/> |
| supervisord | `BSD-3-Clause`-style (`repoze`) | Agendaless Consulting and contributors | <https://github.com/Supervisor/supervisor> |

The Ruby gem set that Rails pulls in is **not** enumerated (G6); several gems are MIT, some are
LGPL or source-available, and only an SBOM of the image settles it.

### 6.6 `metabase/metabase:v0.65.2`

| Bundled component | Licence (SPDX) | Copyright | Source |
| --- | --- | --- | --- |
| Clojure runtime and application | `EPL-1.0` | Rich Hickey and contributors; Metabase, Inc. for the application | <https://clojure.org/>, <https://github.com/metabase/metabase> |
| Java runtime (OpenJDK-derived) | `GPL-2.0-only` with the Classpath Exception | Oracle / OpenJDK contributors | <https://openjdk.org/> |
| Jetty | `Apache-2.0` or `EPL-2.0` (dual) | Eclipse Foundation | <https://eclipse.dev/jetty/> |
| JDBC drivers | per driver (e.g. `BSD-2-Clause`, `Apache-2.0`, `MIT`) | respective vendors | the driver projects |
| SQLite (`sample-database.sqlite`) | `blessing` (public domain) | D. Richard Hipp and contributors | <https://sqlite.org/> — see G5 |

### 6.7 Alpine Linux base — `postgres:18.7-alpine`, `redis:7.4.12-alpine`

| Component | Licence | Copyright | Source |
| --- | --- | --- | --- |
| Alpine Linux | mixed | Alpine Linux Development Team | <https://alpinelinux.org/> |
| musl libc | `MIT` | Rich Felker and contributors | <https://musl.libc.org/> |
| BusyBox | `GPL-2.0-only` | Erik Andersen, Rob Landley and contributors | <https://busybox.net/> |

BusyBox is `GPL-2.0-only`; it is therefore the second component (after MariaDB) that must stay behind
a process boundary — §7.2. Alpine's per-package licence metadata lives in `/lib/apk/db/installed`.

### 6.8 Debian packages installed by `upstream/D07-01/Dockerfile`

The build installs development libraries and `unzip`; the runtime `.so` files that PHP extensions link
against remain in the image. Each is under its own licence:

| Package | Licence (SPDX) | Copyright |
| --- | --- | --- |
| `libfreetype6-dev` | `FTL` or `GPL-2.0-only` (dual) | The FreeType Project |
| `libjpeg-dev` | `IJG` | Independent JPEG Group |
| `libpng-dev` | `libpng-2.0` | PNG Reference Library authors |
| `libzip-dev` | `BSD-3-Clause` | Dieter Baron and Thomas Klausner |
| `libldap2-dev` | `OpenLDAP-2.8` | The OpenLDAP Foundation |
| `libicu-dev` | `Unicode-3.0` | Unicode, Inc. and contributors |
| `unzip` | `Info-ZIP` | Info-ZIP authors |

Debian ships the authoritative text for each of these at
`/usr/share/doc/<package>/copyright` inside the image; that directory, not this table, is the notice
that must travel with a redistribution.

### 6.9 Enumerating what is inside an image (the honest fix for G6)

Every `inherited` row above can be replaced by an exact inventory with one of these commands, run on
a host that already holds the image. Use `temp/` for the output; it is git-ignored.

```powershell
# 1. Package-level SBOM (Docker Desktop / BuildKit; Syft is the portable alternative)
docker sbom espocrm/espocrm:10.1.4 --format spdx-json | Out-File -Encoding utf8 temp/sbom-espocrm-10.1.4.json

# 2. The licence files that actually travel inside the image
docker run --rm --entrypoint sh espocrm/espocrm:10.1.4 -c 'ls -1 /usr/share/doc | head -50; cat /LICENSE 2>/dev/null | head -20'

# 3. Aggregated Debian/Ubuntu copyright notices
docker run --rm --entrypoint sh frappe/erpnext:v16.53.0 -c 'grep -h "^License:" /usr/share/doc/*/copyright 2>/dev/null | sort | uniq -c | sort -rn | head -30'

# 4. Alpine package licence metadata
docker run --rm --entrypoint sh postgres:18.7-alpine -c 'grep -E "^P:|^L:" /lib/apk/db/installed | paste - - | sort -u | head -50'

# 5. OCI labels a product declares about itself
docker inspect --format '{{json .Config.Labels}}' openproject/openproject:17.9.6
```

When an SBOM reveals a licence family with an obligation that is not already covered in §7 or §8,
add a row here **and** state the obligation; when it reveals only permissive licences, a single line
naming the family is enough.

---

## 7. Licence compatibility and obligations

### 7.1 The platform's own licence

This repository is `GPL-3.0-only`. `LICENSE` contains the verbatim GNU GPL v3 text; upstream records
the choice as `GPL-3.0-only` (`ADR-BE-19`), and the two built image labels state
`GPL-3.0-only AND <product reference>`. Everything this repository authors — the Compose
definitions, the catalog records, the bootstrap scripts, the `IMAGE-LICENCES.md` files and this
document — is under that licence.

### 7.2 `GPL-3.0-only` versus `GPL-2.0-only` — the rule §7 exists for

`GPL-2.0-only` code may not be combined with `GPL-3.0-only` code into a single work. Every
`GPL-2.0-only` component in this repository is therefore kept at arm's length:

| Component | Why it is safe as deployed | What would break it |
| --- | --- | --- |
| MariaDB 11.4 | separate container, reached over TCP; no shared address space, no linked library | linking `libmariadb`/`libmysqlclient` into platform or product code, or shipping it inside the platform's own image |
| BusyBox (Alpine) | part of the sidecar image's userland, never mounted into another container | exporting `/bin` from the Postgres/Alpine image into another image |

The same test applies to any product whose identifier is currently `GPL-3.0` but unverified (G1, G2):
if Mautic or OrangeHRM turns out to be `GPL-2.0-only`, its thin wrapper image becomes an
incompatible combination, and the wrapper must be split out of the product's layer stack rather than
shipped as one image. Both `IMAGE-LICENCES.md` files say exactly this, and both point here.

This is an engineering rule derived from the licences, not a legal opinion. Where the analysis
matters commercially, it needs review.

### 7.3 AGPL-3.0 products (EspoCRM, Metabase)

Both are deployed as separate services behind the edge. AGPL-3.0 §13 obliges whoever runs a
**modified** version to offer the Corresponding Source to users interacting with it over a network.
This repository does not patch either product, so no additional source obligation arises from a
stock deployment; an operator who applies an in-product customisation (a plugin, a patched image)
takes that obligation on themselves.

### 7.4 GPL-3.0 products (Mautic, ERPNext/Frappe, OrangeHRM, OpenProject)

Conveying any of these images — `docker save`, an exported appliance, a copy of the host's image
store — triggers GPL-3.0 §6: the Corresponding Source, or a written offer valid for three years,
must accompany the object code. Building the image on the operator's own host and running it there
is not conveying, and the platform never pushes these tags to a registry (`pull_policy: never` for the
two locally built images). The export path is therefore the only place the obligation arises, which
makes `docker save` a licensing step as well as a backup step.

### 7.5 Redis is the one component whose licence is not settled by this repository

`redis:7-alpine` was not a fixed artefact. Redis 7.4 moved from `BSD-3-Clause` to a dual
`RSALv2`/`SSPL-1.0` licence, and Redis 8 added `AGPL-3.0-only` as an option. A deployment that
resolved the tag in 2024 and one that resolves it today may therefore have been running under
different terms with no change in this repository. The Compose file now pins
`redis:7.4.12-alpine@sha256:7f7d30b7…`, so both the family and the artefact are fixed at
`RSALv2`/`SSPL-1.0` and a deployment can no longer drift with the tag. What stays open (G3) is the
licence policy, not the pin: re-pin `7.2.x` for a `BSD-3-Clause` baseline or `8.x` for
`AGPL-3.0-only` as a reviewed change.

### 7.6 Notices must travel with the artefact

- Do not strip `/usr/share/doc/*/copyright`, `/LICENSE`, `COPYING` or `NOTICE` files from an image.
- Keep the `org.opencontainers.image.licenses` label accurate; it is the only licence statement a
  `docker inspect` can reach. Both locally built images currently carry a `LicenseRef-…` placeholder
  for an unverified product licence — replace it, do not delete it.
- If a component's licence requires attribution in user-facing documentation and the deployment has
  such documentation, reproduce the notice there as well.

---

## 8. Required attribution and notice texts

The full licence texts are **not** reproduced here: they travel inside each image and in each
upstream repository, and copying them creates a second copy that silently goes stale. What follows is
what must appear, and where the authoritative text lives.

### 8.1 GPL family (`AGPL-3.0-only`, `GPL-3.0-only`, `GPL-2.0-only`)

Required with any redistribution:

1. the complete licence text — from the image or the upstream repository named in the entry above;
2. the copyright notices of the component, unmodified;
3. for a modified version, a prominent statement of the change and its date (GPL-3.0 §5);
4. the Corresponding Source or a written offer (GPL-3.0 §6, AGPL-3.0 §13 for network use).

No short "licensed under…" line substitutes for items 1 and 2.

### 8.2 PostgreSQL Licence

PostgreSQL is the one component whose notice is short enough to reproduce in full, and whose licence
requires the paragraphs to appear in all copies. Take the exact text — including the year range,
which advances annually — from `/usr/share/doc/postgresql/copyright` in the image, or from
<https://www.postgresql.org/about/licence/>. The operative content is:

```text
PostgreSQL Database Management System
(formerly known as Postgres, then as Postgres95)

Portions Copyright (c) 1996-<year>, The PostgreSQL Global Development Group
Portions Copyright (c) 1994, The Regents of the University of California

Permission to use, copy, modify, and distribute this software and its
documentation for any purpose, without fee, and without a written agreement
is hereby granted, provided that the above copyright notice and this paragraph
and the following two paragraphs appear in all copies.

IN NO EVENT SHALL THE UNIVERSITY OF CALIFORNIA BE LIABLE TO ANY PARTY FOR
DIRECT, INDIRECT, SPECIAL, INCIDENTAL, OR CONSEQUENTIAL DAMAGES, INCLUDING
LOST PROFITS, ARISING OUT OF THE USE OF THIS SOFTWARE AND ITS DOCUMENTATION,
EVEN IF THE UNIVERSITY OF CALIFORNIA HAS BEEN ADVISED OF THE POSSIBILITY OF
SUCH DAMAGE.

THE UNIVERSITY OF CALIFORNIA SPECIFICALLY DISCLAIMS ANY WARRANTIES,
INCLUDING, BUT NOT LIMITED TO, THE IMPLIED WARRANTIES OF MERCHANTABILITY
AND FITNESS FOR A PARTICULAR PURPOSE. THE SOFTWARE PROVIDED HEREUNDER IS
ON AN "AS IS" BASIS, AND THE UNIVERSITY OF CALIFORNIA HAS NO OBLIGATIONS TO
PROVIDE MAINTENANCE, SUPPORT, UPDATES, ENHANCEMENTS, OR MODIFICATIONS.
```

### 8.3 PHP Licence 3.01

PHP-3.01 requires the credit line to appear in documentation or other materials shipped with a
product that includes PHP:

```text
This product includes PHP software, freely available from
<http://www.php.net/software/>.
```

The licence also carries BSD-style conditions (retain the copyright notice, do not use "PHP" as a
product name without permission). Authoritative text: <https://www.php.net/license/3_01.txt>.

### 8.4 Apache-2.0 (Apache HTTP Server, Jetty, parts of Metabase)

Ship a copy of the Apache License 2.0, retain any upstream `NOTICE` file verbatim, and mark modified
files. No fixed attribution string is required; the `NOTICE` file is what must not be lost.

### 8.5 BSD, MIT, IJG, libpng, OpenLDAP, Unicode, Info-ZIP, `blessing`

The obligation is uniform and short: **retain the copyright notice and the permission text** in
source and binary redistributions, and do not use the authors' names to endorse a derived product.
The authoritative text for each Debian/Alpine package is the per-package `copyright` file inside the
image (§6.8, §6.9). SQLite's `blessing` imposes no condition at all.

### 8.6 `RSALv2` / `SSPL-1.0` (Redis, if the pinned release is 7.4+)

Required with redistribution: the licence text and the copyright notice. The substantive restriction
is operational: do not offer the *functionality of Redis* to third parties as a service, and do not
publish a modified version as such a service. Using Redis internally as ERPNext's cache and queue is
inside the grant. Text: <https://redis.io/legal/licenses/>.

---

## 9. Maintenance

### 9.1 What forces an update to this file

| Trigger | Sections to touch |
| --- | --- |
| An image tag, digest or version changes in `catalog/<id>.json` or `upstream/<id>/docker-compose.yml` | §2.1 row, the product entry in §4 or §5, §3 map |
| A `Dockerfile` base image changes | §6 |
| A new catalog entry is added | §2.1, §3, §4, and §6 if it builds locally |
| A verification gap in §2.2 is closed | move the row, flip the Evidence cell, delete the placeholder licence reference |
| A licence text is read out of an image and differs from the catalog | fix `catalog/<id>.json` first, then §4/§5, then §7.2 if the family changed |
| A new SBOM is generated | §6.9 output reviewed; add a row per licence family with an obligation |

### 9.2 Procedure for a version bump (PowerShell, from the repository root)

```powershell
# 1. Read the new licence out of the artefact, not the repository
docker run --rm --entrypoint sh <image>:<tag> -c 'cat /LICENSE /COPYING 2>/dev/null | head -40'

# 2. Confirm the identifier the catalog claims and compare with the upstream repository
Select-String -Path catalog\*\*.json -Pattern 'licenseId|imageRef|imageDigest'

# 3. Re-check the image's own licence labels
docker inspect --format '{{index .Config.Labels "org.opencontainers.image.licenses"}}' <image>:<tag>

# 4. Update this file in the same commit as the catalog change
git --no-pager diff -- THIRD-PARTY-NOTICES.md catalog upstream
```

### 9.3 Review checklist

- [ ] Every image named in an `upstream/*/docker-compose.yml` appears in §2.1.
- [ ] Every `FROM` in an `upstream/*/Dockerfile` appears in §6.
- [ ] Every row states a full SPDX expression, not the catalog's short form.
- [ ] Every `unverified` / `tag-dependent` row names the gap in §2.2 and the action that closes it.
- [ ] No placeholder (`LicenseRef-…`, `<year>`, "TBD") survives a release without a §2.2 entry.
- [ ] §7.2 still holds: no `GPL-2.0-only` component is linked into GPL-3.0-only code.
- [ ] The document version and "Last reviewed" date are bumped.

### 9.4 Document control

| Version | Date | Change | Reviewer |
| --- | --- | --- | --- |
| 1.0 | 2026-11-25 | Initial inventory: six catalog applications, four sidecar images, six base-image/OS groups, compatibility rules, notices and maintenance procedure | platform engineering |
| 1.1 | 2026-11-26 | D01-01 moved to EspoCRM 10.1.4 (§2.1, §3, §4.1, §6.3, §6.9); product image now pinned by tag **and** digest; base image read out of 10.1.4 and found unchanged from 10.0.11 (PHP 8.4 / Debian 13) | platform engineering |
