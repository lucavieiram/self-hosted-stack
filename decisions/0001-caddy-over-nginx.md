# 1. Caddy over nginx

**Accepted, in production. 2026-07.**

## Context

Was running nginx + certbot. The origin is locked to Cloudflare IP ranges, so HTTP-01
can't reach it — the ACME server isn't Cloudflare and gets dropped. Issuance had to move
to DNS-01.

## Decision

Caddy built with the Cloudflare DNS plugin:

```dockerfile
FROM caddy:2-builder AS builder
RUN xcaddy build --with github.com/caddy-dns/cloudflare
FROM caddy:2
COPY --from=builder /usr/bin/caddy /usr/bin/caddy
```

TLS for every site, reused via a snippet:

```
(cftls) {
    tls {
        dns cloudflare {env.CF_API_TOKEN}
        resolvers 1.1.1.1 8.8.8.8
    }
}
```

Two sites plus security headers: ~15 lines. The nginx version was ~120 across several
files, plus a certbot timer and a renewal hook that could break silently.

## Rejected

- **nginx + certbot DNS plugin.** Works. Keeps certificate lifecycle as a separate moving
  part I have to monitor.
- **Traefik.** Label-based discovery pays off when containers churn. I add a service every
  few weeks by hand.

## Cost

Caddy is rare in industry — this is not the employable choice, and I keep nginx literacy
separately. Custom build means I own image rebuilds. Config reload is all-or-nothing: one
syntax error takes down every site.

## Verified

Certificate issued over DNS-01 against a Cloudflare-locked origin. Both sites serving.
HSTS and `X-Content-Type-Options` present.
