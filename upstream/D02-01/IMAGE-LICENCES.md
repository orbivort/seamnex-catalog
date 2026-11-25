# Licences of the image this tree builds

This file travels **inside** the image (`/usr/share/doc/seamnex/mautic-licences.md`) and is named by its
`org.opencontainers.image.licenses` label. It exists because a container image is a distribution:
whoever pulls it receives three different bodies of code under three different sets of terms, and a
label that names only one of them is a claim the recipient cannot check.

It is an inventory, not legal advice, and it is deliberately explicit about the one thing this tree has
not verified.

| Layer | What it is | Terms | Status |
|---|---|---|---|
| Platform additions | `seamnex-bootstrap.sh`, this file | GPL-3.0-only (this repository's licence, `ADR-BE-19`) | Known |
| Base image | `mautic/mautic:7.2.2-apache` — PHP, Apache HTTP Server, Debian packages | PHP-3.01, Apache-2.0, and the mixed (predominantly MIT/BSD/LGPL) set of Debian | Known, inherited |
| Mautic 7.2.2 | the application shipped inside the base image | The project declares GPL-3.0; the product's own `LICENSE` has **not** been read out of the image | **Unverified — see below** |

## The unverified part, stated plainly

The catalogue entry records `licenseId: "GPL-3.0"` and `THIRD-PARTY-NOTICES.md` records
`mautic/mautic` at `GPL-3.0` with **`repo`** evidence — the repository's licence file, not the image's.
This tree never fetches the product itself: the base image is pulled and the layer adds only an
entrypoint, so the terms that matter are the ones **inside that image**, and nobody has opened it to
confirm them. Two things must be checked against the artefact before this image is redistributed or
offered as a service:

1. **The identifier and the version.** `GPL-3.0` and `GPL-3.0-only` are not interchangeable here. The
   platform is pinned at `GPL-3.0-only` (`scripts/check-license.mjs`), and a `GPL-2.0-only` product
   **may not be combined** with it (`THIRD-PARTY-NOTICES.md` §7). Read `LICENSE` inside the image. If
   the product turns out to be `GPL-2.0-only`, this image as built is not conveyable, and the render
   set has to separate the platform-authored entrypoint from the product rather than ship them in one
   layer stack.
2. **The bundled component inventory.** The application carries a `vendor/` tree (Composer and, in the
   front-end bundle, npm). Read `composer.json`, `vendor/composer/installed.json` and `LICENSE` inside
   the image, and add a row per licence family that carries an obligation — LGPL and anything
   source-available especially.

The label uses the SPDX custom-reference form `LicenseRef-Mautic-Upstream` for exactly this reason: the
image's own terms are `GPL-3.0-only`, and the product's are a reference to a licence this tree has not
yet read. Replace it with the real identifier in the same commit that records the review.

## Obligations that follow either way

- **Source.** GPL-3.0 §6 applies to the object code in this image, including the platform additions.
  Whoever conveys the image (a `docker save`, an exported appliance, a copy of the host's image store)
  must accompany it with the corresponding source or a written offer. Building it on the operator's own
  host and running it there is not conveying — and this image is never pushed to a registry: it is
  built by the deployment, in the platform's namespace, and `pull_policy: never` keeps it there. The
  export path is therefore the only place the obligation can arise.
- **Notices.** The copyright notices and licence texts of every component above travel with the image
  and are not to be stripped from downstream copies.
- **Modification.** This tree adds files; it does not patch the product. Keep it that way. The
  entrypoint wraps the image's own `/entrypoint.sh` rather than replacing its behaviour, and no file
  shipped by the base image is rewritten. A patched PHP file would make the image a modified version
  under GPL-3.0 §5, which obliges the distributor to state the change and date it.
- **The base image's own terms.** `mautic/mautic` is itself a distribution of Mautic, PHP and Debian.
  Its notices travel in the layers below this one, and this file does not restate them.
