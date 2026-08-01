# 8. The mail server manages its own DNS records

**Accepted, in production. 2026-08.**

## Context

A working mail domain needs a lot of DNS: MX, SPF, DMARC, two DKIM selectors, TLS-RPT,
and a rotating `_acme-challenge` TXT for every certificate renewal. Hand-maintaining that
means the zone drifts from what the server actually does — DKIM keys rotate and the
published key doesn't, or a renewal fails because nobody updated a challenge record.

The ACME requirement forces the issue anyway. HTTP-01 and TLS-ALPN-01 need ports 80 and
443, both owned by the reverse proxy, so the mail server has to use DNS-01 — which means
it needs write access to the zone regardless.

## Decision

Give the mail server a DNS API token scoped to one zone, and let it publish its own
records. Certificate issuance, DKIM key publication and renewal all become unattended.

The token is narrow in three ways at once:

- **Permissions:** DNS write plus zone read. Nothing else — notably not zone settings,
  which can change the TLS mode.
- **Zone:** one domain, not the account.
- **Source IP:** the server's addresses only, so a leaked copy is inert elsewhere.

The important part is that publication is per-record-type, not all-or-nothing. That is
what makes the cutover safe: everything except `mx` goes live first, gets verified from an
external resolver, and only then does the MX record move. Certificates and DKIM were
published and confirmed for an hour before mail flow changed at all.

## Rejected

- **Hand-written records.** What most guides describe. Works once, then drifts, and DKIM
  rotation quietly stops matching.
- **A separate ACME client writing certs to a shared volume.** Couples the mail server's
  restarts to another service's renewal schedule, and it still needs the same DNS
  credential.
- **Terraform or a zone file in git.** Better for a zone edited by humans on a schedule.
  Fights a server that needs to write `_acme-challenge` records several times a year on
  its own initiative.

## Cost

The mail server now holds a credential that can rewrite DNS for the domain — the same
credential that could issue certificates in your name. Scoping and the IP allowlist limit
the blast radius but don't remove it. Two consumers share one token, so rotation touches
both.

There is also less visibility: records appear because a server decided to publish them,
not because someone committed a change. The zone is no longer the source of truth — the
server is.

## Verified

Certificate issued over DNS-01, challenge records created and cleaned up automatically.
All seven records confirmed from an external resolver, including forward-confirmed reverse
DNS. Inbound mail from a major provider passed SPF, DKIM and DMARC; outbound was accepted
with MTA-STS enforced.

## Gotcha

Changing *which* records should be published updated the config and scheduled no work.
The publication job is queued when the management mode changes, not when its options do —
so toggling the mode off and on was needed to apply it. Verify that a job was scheduled,
not just that the write succeeded.
