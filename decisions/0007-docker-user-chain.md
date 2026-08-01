# 7. Firewall rules belong in DOCKER-USER, not ufw

**Status:** accepted, in production
**Date:** 2026-07

## Context

I wanted the origin to accept web traffic only from Cloudflare's published IP ranges, so
that nobody who discovers the host's address can bypass the edge and hit it directly.

The obvious implementation is a ufw rule. It does nothing.

Docker manages its own iptables rules and inserts them into the `nat` table's
`PREROUTING` chain and the `FORWARD` chain, ahead of where ufw's rules live. A published
container port is DNAT'd to the container before ufw's filtering is ever consulted. The
ufw status output looks exactly as intended and the port is wide open.

I did not learn this from documentation. I found a metrics agent I believed was
firewalled, bound to `*:45876`, and confirmed from an off-host machine that it accepted
connections from the open internet.

## Decision

Rules go in the `DOCKER-USER` chain, which Docker guarantees is traversed before its own
rules, and which Docker does not overwrite.

```
# Allow Cloudflare ranges to reach the published web ports, drop the rest.
# -i eth0 is load-bearing: without it this also matches container egress.
iptables -I DOCKER-USER -i eth0 -p tcp -m multiport --dports 80,443 \
         -m set --match-set cf-ranges src -j RETURN
iptables -A DOCKER-USER -i eth0 -p tcp -m multiport --dports 80,443 -j DROP
```

Services that should never be public are additionally bound to the tailnet address
instead of `0.0.0.0`, so the firewall is the second line rather than the only one. See
decision 2.

## Rejected

**Disabling Docker's iptables management** (`"iptables": false`). Puts the rules back
under my control, and means writing every NAT and forwarding rule for every container by
hand. High effort, easy to get subtly wrong, and it breaks inter-container networking
until you rebuild it correctly.

**Binding everything to localhost and reverse-proxying.** Works well for HTTP and is what
the proxy already does. Does not generalise to non-HTTP services, and does not help when
the thing you need to restrict is the proxy itself.

## Cost

These rules are not managed by ufw, so `ufw status` no longer tells the whole story —
anyone reading the firewall has to know to check two places. They also need to persist
across reboot, which is one more thing to get right in provisioning.

## The mistake inside the fix

My first version of the rule matched destination ports 80 and 443 without specifying an
input interface. `DOCKER-USER` sits in the `FORWARD` chain, which carries *both*
directions of container traffic — so the rule matched every outbound packet from every
container to any web server, and all egress died instantly. Adding `-i eth0` scoped it to
traffic actually arriving from outside.

Two lessons, and the second is the bigger one. A firewall rule needs to state its
direction, not just its port. And a firewall change must be verified from off the host: I
had previously "confirmed" a rule from a shell on the machine itself, which tests a path
the rule was never in.

## Related

Discovered again later that `ufw` had been removed from the box entirely — package state
`rc`, `INPUT` policy `ACCEPT` — while every prior check had reported the rules as fine.
Verify the enforcement point exists before trusting what it reports.
