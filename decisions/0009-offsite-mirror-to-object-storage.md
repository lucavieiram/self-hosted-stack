# 9. Mirror the backup repo to object storage

**Accepted, in production. 2026-08.**

## Context

[Decision 4](0004-restic-with-db-dumps.md) left one thing open: the snapshots lived on
the machine they protected. That covers a bad deploy or a dropped table. It covers
nothing that takes the host with it — a provider incident, a wiped disk, an attacker who
gets root and reaches the repo.

A backup that shares a failure domain with its source is a convenience feature, not a
backup.

## Decision

A second restic repository in S3-compatible object storage, mirrored nightly with
`restic copy`.

```bash
# At init only. Without this, the second repo chunks differently and every copy
# re-uploads content the repo already holds.
restic init --repo "$OFFSITE" --copy-chunker-params

# Nightly, after the local snapshot is already consistent
restic copy --from-repo "$LOCAL" --from-password-file "$PWFILE"
restic forget --tag auto --keep-daily 7 --keep-weekly 4 --keep-monthly 6 --prune
```

Three things this buys, in order of how much they matter:

**`--copy-chunker-params` at init.** restic splits files into content-defined chunks
using per-repository parameters. A repo initialised without inheriting them produces
different chunk boundaries for identical data, so `copy` uploads a full second copy
forever. This flag is only honoured at `init` — getting it wrong means destroying the
repo and starting over.

**The mirror runs last.** The local snapshot is already written and consistent before
the upload begins. An object storage outage costs the offsite copy for one night. It
cannot cost the backup.

**The whole step is conditional on its credential file existing.** No file, no mirror,
exit 0. The nightly job does not start failing on a host that was never given offsite
credentials.

Provider chosen for zero egress fees. A restore is the one moment the backup has to be
cheap to read — per-GB egress prices exactly the operation you most need to rehearse,
which is how untested restores happen.

## Rejected

- **Sync the repo directory with rclone/rsync.** Copies restic's internal structure with
  no understanding of it. A partial transfer produces a repo that looks present and fails
  at restore. `restic copy` moves snapshots, and the destination is a valid repository
  after each one.
- **Second local disk.** Same building, same provider, same blast radius.
- **A different tool for offsite.** Two formats, two restore procedures, one of them
  never practised.

## Cost

Storage bill scales with retention. A second credential to rotate. The copy adds minutes
to the nightly window on the first run and seconds after — this repo settled at ~700 MiB
stored and a 1.5 MiB incremental.

The real cost is a trap: **the repo password must exist somewhere other than inside the
repo it unlocks.** Escrowing it only in the backup is a circular dependency that stays
invisible until the day the host is gone. Same for the age key that decrypts the
secrets — if it is not in the backup targets, it is not backed up at all. Both belong in
an offline password manager before the offsite copy counts as real.

## Verified

`restic check --read-data-subset=1/10` against the offsite repo: no errors. Then the part
that actually matters — restored a database file from the offsite repo into a scratch
directory and ran `PRAGMA integrity_check`. `ok`, real tables, correct row counts.

Checking the repo proves the repo. Only a restore proves the backup.
