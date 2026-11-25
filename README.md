# Seamnex Catalog

Curated catalog of self-hostable open-source business applications that the Seamnex platform
deploys for small and mid-sized teams (roughly 5–100 users). Each entry pins one product to a
reviewed upstream release and describes everything needed to deploy it reproducibly: runtime image
and digest, licence, resource shape, persistent volumes, integration surface, and a typed
environment declaration.

This repository contains **catalog data and vendored deployment definitions, not platform code**.
The control plane, edge proxy, identity provider and host runtime live in the
[Seamnex platform repository](https://github.com/orbivort/seamnex). Nothing here compiles or runs on
its own except the per-entry Docker Compose files, and those are designed to be driven by the
platform rather than by a host port.

## Repository layout

| Path | Contents |
| --- | --- |
| `catalog/<id>.json` | One product record: version, image reference and digest, licence, stack, resources, volumes, integration contract, owner and verification date. |
| `catalog/env/<id>.json` | The `env-schema/v1` declaration for that record: every variable the deployment reads, with type, constraint, default and override ceiling. |
| `upstream/<id>/` | Vendored deployment source: `docker-compose.yml`, optional `Dockerfile` and first-boot scripts, `SOURCE_REVISION`, `README.md`, and `IMAGE-LICENCES.md` where the image is built locally. |
| `LICENSE` | The GNU GPL v3 text; this repository is licensed under GPL-3.0-only. |
| `THIRD-PARTY-NOTICES.md` | Inventory of every third-party component, its SPDX licence, evidence status and known verification gaps. |
| `CHANGELOG.md` | Notable changes, in Keep a Changelog format. |

## Available products

| Entry | Product | Domain | Stack | Image | Licence | Identity | Port | Resources |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| `D01-01` | EspoCRM 10.0.11 | CRM & Sales Pipeline | PHP / MySQL | `espocrm/espocrm:10.0.11` | AGPL-3.0 | Native OIDC (authorization code + PKCE) | 80 | 1 vCPU / 2 GiB |
| `D02-01` | Mautic 7.2.2 | Marketing & Lead Generation | PHP / MySQL | `seamnex/mautic:7.2.2` (built locally) | GPL-3.0 | Forward-auth gate, product login | 80 | 2 vCPU / 2 GiB |
| `D06-01` | ERPNext 16.53.0 | ERP & Business Operations | Python / MariaDB | `frappe/erpnext:v16.53.0` | GPL-3.0 | Native OIDC (Frappe Social Login Key) | 8080 | 4 vCPU / 8 GiB |
| `D07-01` | OrangeHRM 5.9 | HR & People Operations | PHP / MySQL | `seamnex/orangehrm:5.9` (built locally) | GPL-3.0 | Forward-auth gate, product login | 80 | 2 vCPU / 3 GiB |
| `D09-01` | OpenProject 17.9.6 | Project & Work Management | Ruby / PostgreSQL | `openproject/openproject:17.9.6` | GPL-3.0 | Forward-auth gate, product login | 80 | 4 vCPU / 4 GiB |
| `D11-01` | Metabase 0.65.2 | Data, Reporting & BI | JVM / PostgreSQL | `metabase/metabase:v0.65.2` | AGPL-3.0 | Forward-auth gate, product login | 3000 | 2 vCPU / 4 GiB |

The licence column mirrors the short identifier recorded in each catalog record. The port column is
the container port the platform routes to (`httpPort`); a managed deployment publishes no host port.
Full SPDX expressions, transitive components and evidence status are in `THIRD-PARTY-NOTICES.md`.

Two entries build their own image during deployment and are never pushed:

- `D02-01` — a thin layer over `mautic/mautic:7.2.2-apache` that adds a first-boot installer.
- `D07-01` — OrangeHRM built from the official release archive on `php:8.3-apache-bookworm`, because
  upstream publishes no image.

## Licence and verification

This repository is licensed under **GPL-3.0-only**; the full text is in `LICENSE`. Everything
authored here — catalog records, environment schemas, Compose definitions, bootstrap scripts and
documentation — carries that licence.

Products and container images referenced by the catalog remain under their own licences. Before
redistributing an image (`docker save`, an appliance export, a copied image store), read
`THIRD-PARTY-NOTICES.md`.
