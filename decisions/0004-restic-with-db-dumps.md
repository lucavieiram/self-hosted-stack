# 4. Dump databases before snapshotting

**Accepted, in production. 2026-07.**

## Context

Pointing a backup tool at `/var/lib/docker/volumes` produces a backup that restores
without error and contains a corrupt database.

A live SQLite file copied mid-write can be caught between a page write and the journal
update. InnoDB has the same problem across its tablespace and redo log. The failure
surfaces at restore time.

## Decision

Dump first, snapshot the dumps alongside the volumes.

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

restic for content-addressed dedup, encryption by default, and a documented repository
format that isn't tied to a vendor.

`--single-transaction` matters — without it `mariadb-dump` locks tables and the app stalls
for the length of the dump.

## Rejected

- **Filesystem/block snapshots.** Better answer where available, atomic across the whole
  filesystem. Not available on this storage layer.
- **Stopping containers first.** Consistent, and the client's site is down every night.
- **Provider backups only.** Kept as a second layer. Whole-disk images, no file-level
  restore, no cheap way to test one.

## Cost

The dump step needs DB credentials on the host, so secrets management became a
prerequisite. Dumps cost disk during the run. Small inconsistency window between dump and
snapshot.

## Verified

Restored a file from a snapshot into a scratch dir and compared MD5 against live.
Identical.

**Open:** no offsite copy yet. Snapshots on the machine they protect survive a bad deploy,
not a dead host. That is a single point of failure.
