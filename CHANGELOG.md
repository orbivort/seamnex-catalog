# Changelog

All notable changes to this repository are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and version numbers
follow the `MAJOR.MINOR.PATCH` form of [Semantic Versioning](https://semver.org/spec/v2.0.0.html).
Release tags carry a `catalog-` prefix, and the major component is the release year, so tag
`catalog-2026.11.0` corresponds to version `2026.11.0`.

## [2026.11.1]

### Fixed

- Sidecar images are pinned by tag **and digest** in every vendored Compose file. A deployment of a
  product whose definition referenced `mariadb:11.4` failed at `apply` with
  `(HTTP code 404) no such image — No such image: mariadb:11.4`: a floating alias is not carried by
  every registry mirror the platform pulls through, while the release tag is. The same reasoning was
  applied to the other floating sidecar tags, so a definition no longer depends on registry alias
  state — or on a tag being repointed — to deploy:
  - `D01-01`, `D02-01`, `D06-01`, `D07-01` — `mariadb:11.4` → `mariadb:11.4.13@sha256:a8992b5e5404d11e03c54a31d0c1bdea3fd116d283aa6d1a627551f29045309b`
    (the manifest `11.4` already resolved to on 2026-11-25);
  - `D06-01` — `redis:7-alpine` → `redis:7.4.12-alpine@sha256:7f7d30b7fdd09467bac58198a1f166dd69789d348d4fe859cccb20e0cf287fbe`
    (both Redis services);
  - `D11-01` — `postgres:18-alpine` → `postgres:18.7-alpine@sha256:0115267ab5a509022df9319c77be365cef335adb71556c75f60f5360ee247e71`.
  Each pinned reference is the exact manifest its floating tag resolved to at the time of the change,
  so the change buys availability and reproducibility, not a different runtime. The catalog records
  carry no sidecar digest field, so the Compose `image:` references are the authority for these three
  artefacts; `THIRD-PARTY-NOTICES.md` §1.2 and §5 and the per-entry READMEs record the versions and
  digests.

## [2026.11.0]

### Added

- Catalog entries, environment schemas and vendored upstream trees for six products:
  - `D01-01` — EspoCRM 10.0.11 (`espocrm/espocrm:10.0.11`, with a MariaDB 11.4)
  - `D02-01` — Mautic 7.2.2 (`seamnex/mautic:7.2.2`, built from `mautic/mautic:7.2.2-apache`, with a MariaDB 11.4)
  - `D06-01` — ERPNext 16.53.0 (`frappe/erpnext:v16.53.0`, with a MariaDB 11.4 and Redis 7)
  - `D07-01` — OrangeHRM 5.9 (`seamnex/orangehrm:5.9`, built from the SourceForge release archive, with a MariaDB 11.4)
  - `D09-01` — OpenProject 17.9.6 (`openproject/openproject:17.9.6`, pinned by digest)
  - `D11-01` — Metabase 0.65.2 (`metabase/metabase:v0.65.2`, with a PostgreSQL 18)

[Unreleased]: https://github.com/orbivort/seamnex-catalog/compare/catalog-2026.11.0...HEAD
