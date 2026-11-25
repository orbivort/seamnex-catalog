# D02-01 Mautic 7.2.2 — a thin layer over the official image.
#
# The upstream image (`mautic/mautic:7.2.2-apache`) ships Mautic **uninstalled**: its entrypoint
# validates the environment, waits for MySQL, then starts Apache. Nothing creates the schema, so `/`
# answers with an error page until a human runs `mautic:install` in a shell. A platform that reports
# "ready" on that page has deployed nothing an employee can use, and the catalog entry already
# declares every value the installer needs.
#
# The functional change is the entrypoint: `seamnex-bootstrap.sh` installs once, then delegates to the
# image's own `/entrypoint.sh`, so the volume checks, migrations, role dispatch and Apache startup
# remain upstream's code rather than a copy of it. The second change is documentation, not behaviour:
# the image carries the licence inventory that its `org.opencontainers.image.licenses` label names, so
# a recipient of a `docker save` can read what they received without leaving the image.
#
# Pinned by tag (the compose file also carries the image name and declares this build); the catalog
# entry records the id the reference host built. The executor builds it in the deployment's `build`
# stage through BuildKit (ADR-BE-40), so no host-side `docker build` is required:
#
#   platform deployment (automatic)   or   docker build -t seamnex/mautic:7.2.2 .
#
# The manual form runs from this directory, which is the build context the renderer rewrites to the
# absolute resolved source tree. It deliberately names no repository path: the tree moves, the tag does
# not.
FROM mautic/mautic:7.2.2-apache

COPY seamnex-bootstrap.sh /usr/local/bin/seamnex-bootstrap.sh
COPY IMAGE-LICENCES.md /usr/share/doc/seamnex/mautic-licences.md
RUN chmod 0755 /usr/local/bin/seamnex-bootstrap.sh

LABEL org.opencontainers.image.title="Mautic 7.2.2 (Seamnex render set)" \
	org.opencontainers.image.description="Mautic wrapped by the platform's vendored render set for D02-01; the tag exists only in the platform's own namespace, is built on the host and is never pushed." \
	org.opencontainers.image.vendor="Seamnex" \
	org.opencontainers.image.source="https://github.com/orbivort/seamnex" \
	org.opencontainers.image.licenses="GPL-3.0-only AND LicenseRef-Mautic-Upstream" \
	org.opencontainers.image.version="7.2.2"

ENTRYPOINT ["/usr/local/bin/seamnex-bootstrap.sh"]
