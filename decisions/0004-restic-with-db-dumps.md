# 4. restic, with database dumps taken before the snapshot

**Status:** accepted, in production
**Date:** 2026-07

## Context

The first instinct is to point a backup tool at `/var/lib/docker/volumes` and call it
done. For anything holding a database, that produces a backup which *restores without
error* and contains a corrupt database.

A live SQLite file copied mid-write can be captured between a page write and the journal
update. InnoDB has the same problem across its tablespace and redo log. The copy is a
snapshot of a moving thing, and the damage does not surface until the day you actually
need it — which is the worst possible day to discover it.

## Decision

Dump first, then snapshot the dumps alongside the volumes.

```bash
# SQLite: .backup is transactionally consistent against a live database
docker run --rm -v app_db:/v:ro -v "$DUMPS":/out alpine sh -c \
  'apk add --no-cache sqlite && sqlite3 /v/database.sqlite ".backup /out/app.sqlite"'

# MariaDB: single transaction, so no table locks and a consistent read view
docker exec db sh -c "exec mariadb-dump -uroot -p\"\$PW\" \
  --all-databases --single-transaction --quick" > "$DUMPS/all.sql"

restic backup --tag auto --exclude-caches "${TARGETS[@]}"
restic forget --tag auto --keep-daily 7 --keep-weekly 4 --keep-monthly 6 --prune
```

restic specifically for content-addressed deduplication, encryption at rest by default,
and the fact that its repository format is well documented and not tied to a vendor.

`--single-transaction` matters: without it, `mariadb-dump` locks tables and the
application stalls for the duration of the dump.

## Rejected

**Filesystem or block-level snapshots.** Genuinely correct, and the better answer where
available — a snapshot is atomic across the whole filesystem. Not available on this VPS's
storage layer, so the dump step does the same job at the application level.

**Stopping containers before backing up.** Guarantees consistency, and means the client
website is down every night. Not acceptable for something someone else depends on.

**Provider backups only.** Kept as a second layer, but they are whole-disk images with no
file-level restore and no way to test a restore without provisioning a machine. A backup
you cannot cheaply test is a backup you do not know you have.

## Cost

The dump step needs database credentials on the host, which means secrets management is
now a prerequisite rather than a nice-to-have. Dumps also cost disk during the run, and
the window between dump and snapshot is a small inconsistency I accept.

## Verification

The part that makes this real: a file was restored out of a snapshot into a scratch
directory and its MD5 compared against the live copy. Identical. `restic check` passing
proves the repository is intact; it does not prove the data restores, and only one of
those is the thing you actually care about.

Still open: the offsite copy. Snapshots living on the same machine they protect are not
a backup — they survive a bad deploy, not a dead host. Object storage in another provider
is the next step, and until it exists this system has a single point of failure I can
name.
