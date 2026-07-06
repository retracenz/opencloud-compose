# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

Deployment configuration only — a collection of Docker Compose files that stand up
[OpenCloud](https://docs.opencloud.eu) and its optional companion services. There is no
application source code, no build step, and no test suite. "Working in this repo" means
editing YAML compose files, the `config/` files they mount, and `.env.example`.

## Core architecture: modular compose layering

`docker-compose.yml` defines only the base `opencloud` service, the `opencloud-net` network,
and the two default volumes. Everything else (reverse proxy, web office, identity, storage,
search, monitoring, antivirus, CalDAV/CardDAV) lives in a category subdirectory and is layered
on top using Docker Compose's multi-file merge.

A running stack is assembled by picking **one file from the relevant categories** and passing
them in order via `-f` flags or the `COMPOSE_FILE` variable in `.env` (colon-separated). Later
files override earlier ones.

Categories and their directories:
- **base** — `docker-compose.yml` (always first)
- **reverse proxy** — `traefik/` (built-in Traefik) **or** `external-proxy/` (you run your own proxy). Mutually exclusive.
- **web office** — `weboffice/collabora.yml` **or** `weboffice/euro-office.yml`. Mutually exclusive (both use the same in-process `collaboration`/WOPI service).
- **identity** — `idm/ldap-keycloak.yml` (bundled Keycloak + OpenLDAP), `idm/external-idp.yml`, or `idm/external-authelia.yml`. Default (no file) uses OpenCloud's built-in IDM/IDP.
- **storage** — `storage/decomposeds3.yml` for S3/MinIO backend (default is local `decomposed` driver)
- **search** — `search/tika.yml` (full-text extraction) or `search/opensearch.yml`
- **other** — `monitoring/`, `radicale/`, `antivirus/clamav.yml`
- **testing helpers** — `testing/` (e.g. phpLDAPadmin); not for production

**Key merge convention:** most module files contain an `opencloud:` service block that adds
only `environment:` keys or `labels:` — these merge into the base service. `traefik/<name>.yml`
files add the Traefik router labels that the matching `weboffice`/`idm` module needs, so they
are paired (e.g. `weboffice/collabora.yml` goes with `traefik/collabora.yml`). Under an
external proxy you use `external-proxy/<name>.yml` instead, which exposes ports rather than
adding labels.

Example full stack (Collabora + Keycloak/LDAP behind Traefik):
```
COMPOSE_FILE=docker-compose.yml:weboffice/collabora.yml:idm/ldap-keycloak.yml:traefik/opencloud.yml:traefik/collabora.yml:traefik/ldap-keycloak.yml
```

`compose-dokploy.yml` is a separate top-level entrypoint (uses `include:`) for Dokploy-managed deployments.

## Configuration model

- **`.env`** drives everything. It does not exist in git — copy `.env.example` (the fully
  documented template) to `.env`. Nearly every value in the compose files is
  `${VAR:-default}`, so defaults target the local `*.opencloud.test` domains.
- **`config/`** holds files mounted into containers: `config/opencloud/` (csp.yaml, proxy.yaml,
  banned-password-list.txt, and the `apps/` mount), `config/keycloak/` (realm definitions in
  `*.dist.json`, OIDC client JSON, custom login theme), `config/ldap/` (LDIF seed files, ACL
  init), `config/traefik/` (entrypoint override + `dynamic/` for custom TLS certs).
- `INITIAL_ADMIN_PASSWORD` matters only with the built-in IDM and only on **first** start —
  it is ignored on later runs.

## Common commands

```bash
cp .env.example .env                       # first-time setup; then edit COMPOSE_FILE + secrets
docker compose up -d                       # start stack defined by COMPOSE_FILE in .env
docker compose -f docker-compose.yml -f traefik/opencloud.yml up -d   # or specify layers explicitly
docker compose down                        # stop
docker compose logs -f opencloud           # tail a service
docker compose config                      # render the merged config (use to verify layering)
docker network create opencloud-net        # required once before using monitoring/monitoring.yml (external network)
```

There is no linter or test runner. `docker compose config` is the closest thing to validation —
run it to confirm a set of layered files merges into what you expect.

## Conventions when editing

- **Image tags** are managed by Renovate (`renovate.json`). Pinned images that Renovate should
  bump carry a `# renovate: depName=...` comment on the line above the `image:` line — preserve
  that pattern. Patch updates auto-merge; `stable-4.0` branch is pinned to no major/minor bumps.
- Keep new tunable values as `${VAR:-default}` and document them in `.env.example` and the
  variables table in `README.md`.
- All services must join the `opencloud-net` network.
- `traefik/` label blocks and `external-proxy/` port blocks are alternatives for the same module —
  when adding a new proxied service, provide both.
