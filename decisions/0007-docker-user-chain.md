# 7. Firewall in DOCKER-USER, not ufw

**Accepted, in production. 2026-07.**

## Context

I wanted the origin to accept web traffic from Cloudflare ranges only, so discovering the
host's address doesn't let you bypass the edge.

The obvious ufw rule does nothing. Docker writes its own rules into `nat` `PREROUTING` and
`FORWARD`, ahead of ufw's. A published port is DNAT'd to the container before ufw is ever
consulted. `ufw status` looks correct and the port is open.

I didn't learn this from docs. I found a metrics agent I believed was firewalled, bound to
`*:45876`, and confirmed from off-host that it accepted connections from the open
internet.

## Decision

Rules go in `DOCKER-USER`, which Docker traverses first and doesn't overwrite.

```
# Without -i eth0 this also matches container egress.
iptables -I DOCKER-USER -i eth0 -p tcp -m multiport --dports 80,443 \
         -m set --match-set cf-ranges src -j RETURN
iptables -A DOCKER-USER -i eth0 -p tcp -m multiport --dports 80,443 -j DROP
```

Services that should never be public also bind to the tailnet address instead of
`0.0.0.0`, so this is the second line, not the only one. See decision 2.

## Rejected

- **`"iptables": false`.** Returns control, and means hand-writing every NAT and forwarding
  rule for every container. Easy to get subtly wrong, breaks inter-container networking
  until rebuilt.
- **Bind localhost, reverse-proxy everything.** Fine for HTTP, which the proxy already
  does. Doesn't generalise to non-HTTP, and doesn't help when the thing to restrict is the
  proxy.

## Cost

`ufw status` no longer tells the whole story — the firewall now lives in two places.
Rules must persist across reboot.

## First version was wrong

First version matched dports 80,443 with no interface. `DOCKER-USER` is in `FORWARD`,
which carries both directions, so it matched every outbound packet from every container to
any web server. All egress died instantly. `-i eth0` scoped it.

A rule has to state its direction, not just its port. And firewall changes must be
verified from off the host — I had previously "confirmed" a rule from a shell on the
machine, which tests a path the rule was never in.

## Related

Later found `ufw` had been removed from the box entirely — package state `rc`, `INPUT`
policy `ACCEPT` — while every prior check reported the rules as fine. Verify the
enforcement point exists before trusting what it reports.
