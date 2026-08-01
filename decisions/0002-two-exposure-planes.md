# 2. Two exposure planes: tailnet private, Cloudflare public

**Status:** accepted, in production
**Date:** 2026-07

## Context

A single host running both a client's public website and my personal file sync, search,
metrics and container control panel. The naive approach gives each service a subdomain
and a password and calls it secured. That leaves five or six login pages on the public
internet, each one a piece of software I now have to patch promptly forever, and each one
a brute-force target.

Most of those services have exactly one user: me. There is no reason for the internet to
be able to reach them at all.

## Decision

Every service is classified public or private *before* it is deployed, and the
classification is enforced by what the container binds to, not by a firewall rule.

**Private** — the container publishes to the tailnet address only:

```yaml
ports:
  - "100.x.x.x:8090:8090"   # WireGuard interface address, not 0.0.0.0
```

There is no listener on the public interface. Nothing to scan, nothing to rate-limit,
nothing to patch urgently. Access requires being on the WireGuard mesh, which is device
authentication rather than a password.

**Public** — reverse-proxied through Caddy, behind Cloudflare, and the origin only
accepts connections from Cloudflare's published IP ranges. Two ports are open: 443, and
25 for inbound mail.

Firewall rules exist as a second line, on the assumption that some day I will bind
something to the wrong interface by accident.

## Rejected

**Everything public with strong passwords and 2FA.** Defensible, and normal practice. But
it converts every self-hosted service into something I must patch on the internet's
schedule instead of mine. A CVE in a file-sync web UI becomes an emergency rather than a
Saturday task.

**A VPN for everything, including the client site.** Not possible — a client's website has
to be reachable by their clients. The split exists precisely because the two categories
have genuinely different requirements.

**Cloudflare Tunnel for all of it.** Attractive, and still on the roadmap for the web
plane. It does not remove the public plane entirely, though: inbound SMTP needs a real
public A record with matching reverse DNS, so the host's address is discoverable anyway
for as long as I run my own mail.

## Cost

If the mesh is down, I cannot reach my own services — accepted, because the failure is
loud and immediate rather than silent. It also creates a dependency on a third-party
coordination service; the data plane is direct WireGuard between devices, but the control
plane is not mine. Self-hosted coordination is the obvious hardening step and is not done
yet.

## Verification

Port scan from an off-host address shows 443 and 25 only. Each private service confirmed
unreachable from a machine that is off the mesh, and reachable from one that is on it.
