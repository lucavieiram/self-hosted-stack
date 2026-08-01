# 3. Docker Compose, not Kubernetes

**Accepted, in production. 2026-07.**

## Context

Six stacks on one 6 GB host: reverse proxy, website, mail, file sync, metasearch,
monitoring. I want infrastructure work as a career, so the pull toward running k8s here
was real.

## Decision

Compose, one directory per stack, plus a small web UI over the compose files so
start/stop doesn't need SSH.

A k3s control plane idles at ~500 MB–1 GB before scheduling any workload of mine. On a
6 GB box targeting 4.5 GB idle, that's a fifth of the budget to schedule onto one node.

Kubernetes solves scheduling across a fleet and self-healing on node loss. I have one
node. If it dies no scheduler saves me — a tested restore does. Running k8s here would be
resume-driven design, and I'd be operating a toy cluster instead of learning the real one.

## Rejected

- **k3s / k0s.** Lower overhead, not zero. Doesn't change the point: no orchestrator gives
  you HA on a single machine.
- **`docker run` + systemd units.** Fewer parts. Compose files diff well in git, which
  matters more.
- **Nomad.** Genuinely good fit at this size. Rejected on ecosystem gravity — when I learn
  a scheduler properly it should be the one the industry runs.

## Cost

No self-healing beyond `restart: unless-stopped`, no rolling deploys, no desired-state
reconciliation. Kubernetes experience has to come from a deliberate multi-node lab I can
break on purpose, not from pretending one VPS is a cluster.

## Verified

Idle memory holds near the 4.5 GB target with all six stacks up.
