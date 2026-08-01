# 6. Self-hosted mail

**Accepted, in production. 2026-07.**

## Context

"Don't self-host email" is good advice. Rejection by the large providers is the default
state, and the feedback loop is bad: mail is accepted, silently filed as spam, and you
find out weeks later.

I did it anyway to understand SMTP, DNS-based authentication and reputation from the
inside. That's a learning decision, not an efficiency one.

## Decision

One modern all-in-one server for SMTP, IMAP and JMAP with an embedded key-value store,
web admin bound to the tailnet. One process instead of Postfix + Dovecot + OpenDKIM +
SpamAssassin — four daemons, four config languages, four failure modes, for one mailbox.

Sequencing was the real decision. MX moves last, so a mistake at any earlier step costs
nothing:

1. Reverse DNS set, forward-confirmed
2. Server up, TLS working, local delivery proven
3. SPF, DKIM, DMARC published and verified from an external resolver
4. **Then** MX

Moving MX earlier means debugging against live mail.

## Rejected

- **Managed forwarding.** What I ran before, and what I'd recommend to anyone whose goal is
  working email. Zero deliverability risk, teaches nothing about the protocol.
- **Postfix stack.** What most production mail runs on. Rejected on operational surface for
  one mailbox — but it's the rejection I'd revisit, because the industry familiarity is
  worth something.
- **Workspace / Fastmail.** Right answer for a business.

## Cost

Deliverability is ongoing, not setup. Reputation on a fresh IP starts at zero. Port 25
must be public, so the host's address is discoverable no matter what else I do — this is
why hiding the origin isn't achievable while I run mail. Host down means bounces.

## Notes

Debian ships an MTA enabled by default; it already owns port 25.

My first rDNS check came back valid from a cached answer and was wrong. `dig @1.1.1.1`
gave the truth. When the DNS record is the thing being proven, a cache proves nothing.

Mail data and DKIM keys started on anonymous volumes. A recreate would have orphaned them
and generated new keys, breaking every signature already published.
