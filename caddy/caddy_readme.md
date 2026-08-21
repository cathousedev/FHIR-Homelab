# Caddy: Multi-Node Routing (Zion / Yosemite / Yellowstone)

## Topology

- **Zion** — edge node. Holds the wildcard TLS cert for `*.cathousedev.com` via `acme_dns cloudflare`. Every public subdomain is defined here as a `@matcher` + `handle` block inside the single `*.cathousedev.com { }` site block, then reverse-proxied to the internal node that actually runs the service.
- **Yosemite** (192.168.1.10) and **Yellowstone** (192.168.1.11) — internal nodes. Each runs its own local Caddy instance with `auto_https off`, since Zion already terminates TLS. These just route `http://` traffic to the right container by service name.

## Zion (edge) Caddyfile pattern

All subdomains live inside the one wildcard block, using matcher + handle — **not** standalone site address blocks nested inside another block (that throws `unrecognized directive`):

```caddyfile
*.cathousedev.com {
    tls {
        dns cloudflare {env.CLOUDFLARE_API_TOKEN}
        resolvers 1.1.1.1
    }

    @aidbox host aidbox.cathousedev.com
    handle @aidbox {
        reverse_proxy 192.168.1.10:80
    }

    @emr host emr.cathousedev.com
    handle @emr {
        reverse_proxy 192.168.1.11:80
    }

    handle {
        respond "Unknown host" 404
    }
}
```

**Gotcha:** `aidbox.cathousedev.com { reverse_proxy ... }` written as its own block *inside* the wildcard block is invalid — Caddy tries to parse the hostname as a directive. Always use `@name host <domain>` + `handle @name { }` for anything nested inside `*.cathousedev.com { }`.

## Internal node Caddyfile pattern (Yosemite / Yellowstone)

Each internal node gets its own Caddyfile (not nested — top-level site blocks are fine here since there's no wildcard wrapper):

```caddyfile
{
    auto_https off
}

http://aidbox.cathousedev.com {
    reverse_proxy aidbox:8080
}

http://emr.cathousedev.com {
    reverse_proxy emr:3000
}

http://yosemite.cathousedev.com {
    respond "Welcome to Yosemite Server" 200
}

http://yellowstone.cathousedev.com {
    respond "Welcome to Yellowstone Server" 200
}
```

Both nodes can share the identical file — routes for services that don't live on that node resolve lazily at request time and just 502 if hit, rather than failing at boot. So `aidbox:8080` sitting unreachable on Yellowstone (no `aidbox` container there) won't crash the container; it'll just error if something actually requests it there.
