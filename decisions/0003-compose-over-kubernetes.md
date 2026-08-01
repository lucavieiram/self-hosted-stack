# 3. Docker Compose plus a web control panel, not Kubernetes

**Status:** accepted, in production
**Date:** 2026-07

## Context

Six stacks on one 6 GB host: reverse proxy, client website, mail, file sync, metasearch,
monitoring. Kubernetes is the thing to put on a CV, and I want infrastructure work as a
career, so the temptation to run it here was real.

## Decision

Docker Compose, one directory per stack, plus a small web UI over the compose files so
starting and stopping a stack does not require SSH.

The budget settles it. A k3s control plane idles at roughly 500 MB to 1 GB before a
single workload of mine is scheduled. On a 6 GB box targeting 4.5 GB idle, that is
something like a fifth of the budget spent on an orchestrator scheduling onto one node —
where "which node" is not a question, because there is only one.

The honest version: Kubernetes solves scheduling across a fleet and self-healing on node
loss. I have one node. If it dies, no scheduler saves me; a tested restore does. Running
k8s here would be resume-driven design, and I would be operating a toy cluster rather
than learning what production Kubernetes actually demands.

## Rejected

**k3s or k0s.** Lightweight distributions genuinely reduce the overhead, but not to zero,
and they do not change the core point: no amount of orchestration provides high
availability on a single machine.

**Plain `docker run` with systemd units.** Fewer moving parts still. Rejected because
compose files are declarative and diff well in git, which matters more to me than
shaving one dependency.

**Nomad.** Lighter than k8s and a genuinely good fit for this size. Rejected on ecosystem
gravity: when I do learn a scheduler properly, the hours are better spent on the one the
industry actually runs.

## Cost

No self-healing beyond `restart: unless-stopped`, no rolling deploys, no declarative
desired-state reconciliation. Kubernetes experience has to come from somewhere else —
a deliberate lab where I can break a real multi-node cluster on purpose, not from
pretending a single VPS is a cluster.

## Verification

Idle memory holds near the 4.5 GB target with all six stacks up.
