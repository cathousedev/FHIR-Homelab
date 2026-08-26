# FHIR-Homelab

A self-hosted FHIR development environment for learning and portfolio work:
**Aidbox** (FHIR server) + **OpenEMR** (EHR), fronted by **Caddy**.

## Architecture

Three-node setup on `cathousedev.com`:

| Node | IP | Role |
|---|---|---|
| **Zion** | — | Edge node — Caddy terminates TLS, routes to internal nodes |
| **Yosemite** | 192.168.1.6 | Runs Aidbox + Postgres |
| **Yellowstone** | 192.168.1.7 | Runs OpenEMR + MySQL |

Cloudflare is used for DNS and Caddy's DNS-01 ACME challenge (wildcard cert
issuance) — it does not proxy traffic; requests terminate directly at Zion.

## Stack

- **Aidbox** — FHIR R4 server
- **OpenEMR** — EHR, used as a source of clinical data
- **Caddy** — reverse proxy / TLS across all three nodes
- **Postgres** / **MySQL** — backing datastores for Aidbox / OpenEMR respectively
- **Synthea** (submodule) — synthetic patient data generation *(not yet wired up)*

## Status

- [x] Aidbox running on Yosemite
- [x] OpenEMR running on Yellowstone
- [x] Caddy routing across all three nodes
- [ ] OpenEMR ↔ Aidbox integration
- [ ] Synthea synthetic data loaded into Aidbox
- [ ] Zion edge Caddyfile committed to this repo (currently node-local, documented as a pattern only)

## Getting started

See [`STARTUP_GUIDE.md`](./STARTUP_GUIDE.md) for full setup instructions,
environment variable reference, and troubleshooting.

Quick version:

```bash
git clone --recurse-submodules https://github.com/cathousedev/FHIR-Homelab.git

# on both Yosemite and Yellowstone
docker network create proxy

# Yosemite
docker compose -f aidbox/compose.yaml up -d

# Yellowstone
docker compose -f openemr/docker-compose.yml up -d

# each node, once its service is up
docker compose -f caddy/compose.yml up -d
```

## Repo layout

```
aidbox/     # Aidbox + Postgres compose, env templates
openemr/    # OpenEMR + MySQL compose, env templates
caddy/      # Shared reverse proxy config (identical on both service nodes)
synthea/    # Submodule — synthetic patient data (not yet integrated)
```

## Why this exists

Built as both a hands-on FHIR learning project and a portfolio piece —
part of a broader push into healthcare informatics/IT. Roadmap items above
(OpenEMR↔Aidbox integration, Synthea-generated test data) are the next steps.
