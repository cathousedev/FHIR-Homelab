## Why this exists

I'm an RN who is a massive linux nerd and have recently become fascinated by FHIR. In addition I am studying for my RHCSA cert and wanted some experiance working with Rocky Linux/ RHEL.

## Stack

- **[Aidbox](https://www.health-samurai.io/aidbox)** — FHIR server
- **[OpenEMR](https://www.open-emr.org/)** — open-source EHR, connected to the FHIR server for realistic clinical data flows
- **[Caddy](https://caddyserver.com/)** — reverse proxy; TLS is handled by the main homelab's Caddy instance (edge node), not by the Caddy config in this repo

This runs as part of a larger self-hosted homelab (Docker Compose + Komodo for orchestration), reachable via a Cloudflare-managed domain. See [`caddy/caddy_readme.md`](./caddy/caddy_readme.md) for the multi-node routing setup.

## Repo layout

```
.
├── aidbox/   # Aidbox FHIR server config
├── caddy/    # Reverse proxy / TLS config
└── openemr/  # OpenEMR config
```

## Running it

1. Clone the repo:
   ```bash
   git clone https://github.com/cathousedev/FHIR-Homelab.git
   cd FHIR-Homelab
   ```
2. This repo does **not** include env files or secrets — you'll need to supply your own for Aidbox and OpenEMR. See their official setup docs:
   - Aidbox: [Run Aidbox locally](https://www.health-samurai.io/docs/aidbox/getting-started/run-aidbox-locally)
   - OpenEMR: [docker-compose.yml (development-easy)](https://github.com/openemr/openemr/blob/master/docker/development-easy/docker-compose.yml)
3. Bring up each service with Docker Compose, e.g.:
   ```bash
   docker compose -f aidbox/docker-compose.yml up -d
   docker compose -f openemr/docker-compose.yml up -d
   docker compose -f caddy/docker-compose.yml up -d
   ```
4. TLS termination and routing are handled by the main homelab's Caddy edge node — see [`caddy/caddy_readme.md`](./caddy/caddy_readme.md) for the full multi-node routing pattern (edge node + internal nodes).

## Roadmap

- [ ] Load synthetic patient data (Synthea) into the aidbox server
- [ ] Initialize and connect Open EMR to Aidbox backend
- [ ] Document integration points between OpenEMR and Aidbox

## Disclaimer

This is a personal lab environment for learning purposes. No real patient data is used.
