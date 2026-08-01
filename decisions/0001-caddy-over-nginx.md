# 1. Caddy over nginx

**Status:** accepted, in production
**Date:** 2026-07

## Context

The stack started on nginx with certbot. That combination works, but the operational
surface is larger than it looks: a server block per site, a separate certbot timer,
renewal hooks that reload nginx, and a failure mode where certificates expire quietly
because the renewal hook broke months earlier and nothing alerted.

Then a hard requirement appeared. The origin is locked to Cloudflare IP ranges, so an
HTTP-01 challenge cannot reach it — the ACME server is not Cloudflare and gets dropped by
the firewall. Certificate issuance had to move to DNS-01.

## Decision

Caddy, built with the Cloudflare DNS plugin via `xcaddy`:

```dockerfile
FROM caddy:2-builder AS builder
RUN xcaddy build --with github.com/caddy-dns/cloudflare
FROM caddy:2
COPY --from=builder /usr/bin/caddy /usr/bin/caddy
```

TLS is then four lines, reused by every site through a snippet:

```
(cftls) {
    tls {
        dns cloudflare {env.CF_API_TOKEN}
        resolvers 1.1.1.1 8.8.8.8
    }
}
```

The whole reverse proxy config, two sites and security headers included, is about
fifteen lines. The nginx equivalent was roughly a hundred and twenty across several
files.

## Rejected

**nginx + certbot with the DNS plugin.** Would have worked. It keeps a config language I
already know and that every employer runs. But it leaves certificate lifecycle as a
separate moving part I have to remember to monitor, and the config stays verbose for a
job that is now genuinely simple.

**Traefik.** Strong at dynamic container discovery through labels. That is a benefit when
containers come and go on their own; on a host where I add a service every few weeks by
hand, it buys little and costs a config model that is harder to read at a glance.

## Cost

Caddy is far less common in industry than nginx, so this is deliberately not the
employable choice — I keep nginx literacy separately rather than pretending Caddy
replaces it. The custom build means I own an image rebuild whenever the plugin or base
image updates; plain `caddy:2` would not do DNS-01. And config reloads are all-or-nothing:
a syntax error takes down every site rather than one.

## Verification

Certificate issued over DNS-01 against a Cloudflare-locked origin, both sites serving,
HSTS and `X-Content-Type-Options` present in response headers.
