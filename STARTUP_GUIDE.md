# FHIR-Homelab — Startup Guide

**Aidbox** (FHIR server) + **OpenEMR** (EHR), routed by **Caddy**. Zion is the
edge node (Caddy, TLS termination); Yosemite and Yellowstone run the services.
Cloudflare is DNS + Caddy's DNS-01 ACME challenge only — not a traffic proxy.

## 1. Prerequisites

- Docker with Compose plugin, git (with submodule support)
- in my case I have **Yosemite** (192.168.1.6) → Aidbox · **Yellowstone** (192.168.1.7) → OpenEMR
- On **both** hosts: `docker network create proxy` (external network, not created automatically)
- DNS/hosts: `aidbox.cathousedev.com` → Yosemite, `emr.cathousedev.com` → Yellowstone
- Zion's edge Caddyfile isn't in this repo — needs `CLOUDFLARE_API_TOKEN` and a wildcard cert set up separately (pattern documented in `caddy/caddy_readme.md`). No edge node? Hit the internal nodes directly (Section 4 ports).

## 2. Clone

```bash
git clone --recurse-submodules https://github.com/cathousedev/FHIR-Homelab.git
```

`synthea` submodule is only for the (not-yet-built) synthetic-data step — skip if you don't need it. No `.env` files are committed; copy from `.example` templates (next section).

## 3. Per-service setup

**Aidbox (Yosemite):** `cp aidbox/{aidbox,postgres}.env.example aidbox/{aidbox,postgres}.env`
- `POSTGRES_PASSWORD` (postgres.env) must **exactly match** `BOX_DB_PASSWORD` (aidbox.env), or Aidbox loops on connection errors
- Also required: `BOX_ADMIN_PASSWORD`, `BOX_ROOT_CLIENT_SECRET`, `BOX_RUNME_UUID` (generate with `uuidgen`)

**OpenEMR (Yellowstone):** `cp openemr/{mysql,openemr}.env.example openemr/{mysql,openemr}.env`
- `MYSQL_ROOT_PASSWORD` (mysql.env) must match `MYSQL_ROOT_PASS` (openemr.env)
- Login is `OE_USER`/`OE_PASS` (defaults `admin`/`pass`) — **not** the MySQL creds

**Caddy:** no env files; same `caddy/` directory runs unmodified on both nodes (routes for services not on that node just 502).

## 4. Bring-up

```bash
# on both Yosemite and Yellowstone
docker network create proxy

# Yosemite
docker compose -f aidbox/compose.yaml up -d

# Yellowstone
docker compose -f openemr/docker-compose.yml up -d

# each node, once its service is up
docker compose -f caddy/compose.yml up -d
```

Aidbox's `depends_on: postgres` has no health condition (postgres has no healthcheck either) — they can race on first boot. If Aidbox logs show Postgres auth failures right after startup, wait for postgres to log "ready to accept connections," then `docker compose -f aidbox/compose.yaml restart aidbox`.

OpenEMR is self-ordering (waits for MySQL healthcheck) but first boot takes a few minutes — normal.

## 5. Verify

```bash
curl -f https://aidbox.cathousedev.com/health
curl -fk https://emr.cathousedev.com/meta/health/readyz
```

If those fail, check locally first: `curl -f http://localhost:8080/health` (Aidbox on Yosemite) or `curl -fk https://localhost:4430/meta/health/readyz` (OpenEMR on Yellowstone) to isolate proxy vs. service issues.

## 6. Common gotchas

- **`proxy network... not found`** → run `docker network create proxy` on that host
- **`env file not found`** → do the `cp` steps in Section 3
- **Port conflicts** (Caddy 80/443, Aidbox 8080, OpenEMR 800/4430) → only the *container-side* port matters for routing, so remap host-side freely
- **Public URLs don't resolve** → DNS must point at the right node IP, and Zion must route to `192.168.1.6:80` / `192.168.1.7:80`
- **502 on a route** → you're hitting the wrong node (Caddyfile is shared but each host only runs its own service)
