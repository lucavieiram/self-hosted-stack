# 6. Self-hosted mail, entered deliberately

**Status:** accepted, in production
**Date:** 2026-07

## Context

Email is the classic "do not self-host this" service, and the advice is sound. Rejection
by the large providers is the default state, not an edge case, and the feedback loop is
brutal: mail is accepted, then silently filed as spam, and you find out weeks later when
someone says they never got it.

I did it anyway, for one reason: the failure modes of SMTP, DNS-based authentication and
reputation are exactly the kind of thing I want to understand from the inside rather than
from a provider's dashboard. That is a learning decision, not an efficiency one, and it
is worth being honest about which is which.

## Decision

A single modern all-in-one server handling SMTP, IMAP and JMAP, with an embedded
key-value store rather than an external database, and a web admin UI bound to the tailnet.

One process instead of the traditional Postfix + Dovecot + OpenDKIM + SpamAssassin
assembly. On a 6 GB host, four daemons with four config languages and four failure modes
is real operational weight for a single-user mailbox.

Sequencing was the important part. Cutting MX over last means a mistake at any earlier
step costs nothing, because mail is still flowing through the previous provider:

1. Reverse DNS set and confirmed forward-confirmed
2. Server up, TLS working, local delivery proven
3. SPF, DKIM, DMARC published and verified from an external resolver
4. **Only then** MX moved

Rushing to step 4 is how people lose mail while debugging.

## Rejected

**Managed email forwarding.** What I ran before, and what I would recommend to anyone
whose goal is working email rather than understanding email. Zero deliverability risk.
It also teaches nothing about the protocol.

**The traditional Postfix stack.** More documentation than anything else in this space,
and unquestionably what most production mail runs on. Rejected on operational surface for
one mailbox — but this is the one rejection I would revisit, because the industry
familiarity has real value.

**Google Workspace or Fastmail.** The right answer for a business. Not the point here.

## Cost

Deliverability is an ongoing responsibility, not a one-time setup: reputation on a fresh
IP starts from nothing and is earned slowly. Port 25 must be open to the internet, which
means the host's address is public and discoverable no matter what else I do — this is
why hiding the origin entirely is not achievable while I run my own mail. If the host is
down, mail bounces rather than queueing somewhere friendly.

## Gotchas

**Something already owns port 25.** Debian ships a mail transfer agent enabled by
default; it has to be disabled before anything else can bind.

**Verify DNS against the authority, not your resolver.** My first reverse-DNS check came
back valid from a cached answer and was wrong. `dig @1.1.1.1` gave the truth. When the
DNS record *is* the thing being proven, querying a cache proves nothing.

**Name the volumes.** Mail data and DKIM signing keys were initially on anonymous volumes
— a container recreate would have orphaned them and generated new keys, breaking every
signature previously published.
