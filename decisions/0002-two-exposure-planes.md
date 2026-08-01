# 2. Tailnet private, Cloudflare public

**Accepted, in production. 2026-07.**

## Context

One host runs a client's public website plus my file sync, search, metrics and container
panel. Giving each a subdomain and a password puts six login pages on the internet, each
one software I must patch on the internet's schedule.

Most of them have one user. There's no reason the internet can reach them.

## Decision

Classify public or private before deploying. Enforce it with the bind address, not a
firewall rule.

Private — bind to the WireGuard address:

```yaml
ports:
  - "100.x.x.x:8090:8090"   # not 0.0.0.0
```

No listener on the public interface. Nothing to scan, no login page to brute-force.
Access is device authentication, not a password.

Public — Caddy behind Cloudflare, origin accepts Cloudflare ranges only. Ports 443 and 25.

Firewall rules are the second line, for the day I bind something to the wrong interface.

## Rejected

- **Everything public with 2FA.** Normal practice. Turns a CVE in a file-sync UI into an
  emergency instead of a Saturday task.
- **VPN for everything.** A client's site has to be reachable by their clients.
- **Cloudflare Tunnel for all of it.** Still on the roadmap for the web plane. Doesn't
  remove the public plane: inbound SMTP needs a public A record with matching rDNS, so the
  host's address is discoverable regardless.

## Cost

Mesh down means I can't reach my own services. Accepted — the failure is loud, not silent.
Control plane is a third party; the data plane is direct WireGuard. Self-hosting the
coordination server is the obvious next step and isn't done.

## Verified

Port scan from off-host shows 443 and 25 only. Each private service unreachable off the
mesh, reachable on it.
