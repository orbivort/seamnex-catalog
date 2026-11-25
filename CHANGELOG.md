# Changelog

All notable changes to this repository are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and version numbers
follow the `MAJOR.MINOR.PATCH` form of [Semantic Versioning](https://semver.org/spec/v2.0.0.html).
Release tags carry a `catalog-` prefix, and the major component is the release year, so tag
`catalog-2026.11.0` corresponds to version `2026.11.0`.

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
