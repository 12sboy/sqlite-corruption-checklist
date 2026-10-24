# SQLite Corruption: First-10-Minutes Checklist

> You just got `database disk image is malformed` (or worse). Don't panic, and
> don't "fix" anything yet. Most SQLite data loss happens *after* the corruption,
> during the recovery attempt. This checklist gets you to a safe starting point
> in ten minutes.

## Minute 0–2: Stop the bleeding

- [ ] **Stop every process that can write to the database.** Not just the app —
      background workers, cron jobs, supervisors that auto-restart. A "harmless"
      monitoring process can still write.
- [ ] **Do not delete anything that looks temporary** (`-wal`, `-shm`, `-journal`).
      Those files may hold your only copy of recent committed data.
- [ ] **Do not reopen the app "to check."** Each open is another chance to write.

## Minute 2–5: Preserve the evidence

- [ ] Note the database path, the time of failure, and the exact error message.
- [ ] Copy **all** related files to a safe location with free space:
  - `app.db` (the main file)
  - `app.db-wal` and `app.db-shm` (if WAL mode — committed data may live *only* here)
  - `app.db-journal` (if rollback-journal mode)
- [ ] Verify the copies are byte-identical in size, then **work only on the copies.**
      The original is now evidence. Touch it again only if you must.

## Minute 5–8: Diagnose before you treat

Run against a **copy**:

```sh
sqlite3 app-copy.db "PRAGMA integrity_check;"
```

- Output `ok` → the file is structurally sound; your problem is elsewhere
  (wrong file, permissions, application bug).
- Lists errors → genuine corruption. Read each line: it tells you *what* is
  damaged (index, table b-tree, freelist), which decides the recovery method.

```sh
sqlite3 app-copy.db "PRAGMA journal_mode;"
```

- `wal` → you must keep the `-wal`/`-shm` pair with the main file. Copying
  the `.db` alone can silently drop committed transactions.
- `delete` → rollback-journal mode; check for a `-journal` file.

## Minute 8–10: Read the error, pick the lane

| Error | Meaning | Lane |
|---|---|---|
| `database disk image is malformed` (11) | B-tree/page damage inside a readable file | `.recover` into a new file, then `integrity_check` the result |
| `file is not a database` (26) | Header destroyed; SQLite can't recognize the file | Recovery tools are unlikely to help — go to backups |
| `database is locked` (5) | Concurrency, not corruption | Check for stale processes / NFS locks; do not "repair" |
| `unable to open database file` (14) | Path/permission problem, not corruption | Verify path, permissions, disk space |

## Before any recovery attempt

- [ ] You have a preserved, untouched copy of the original file set.
- [ ] You know the journal mode and have all companion files.
- [ ] You have somewhere to put the recovered output (it can be *larger* than the source).
- [ ] You have a plan for validating the result (`integrity_check` + spot-checking
      row counts against what you expect) — *before* you put it back in service.
- [ ] You know when to stop: if `.recover` and `.dump` both produce garbage,
      further attempts usually destroy more than they save. That's when a
      professional — or the backup you wish you had — takes over.

## What this checklist doesn't cover

Full recovery procedures: the `.recover` method's silent failures, the `.dump`
ROLLBACK trap, WAL-mode edge cases, and validation in depth. Those need more
than a checklist.

---

*This checklist is a companion to **SQLite Recovery Field Guide: Fix Corrupt
Databases Without Making It Worse** by Le Seine — a field manual of recovery
procedures tested against real corrupted files, with the traps marked and the
limits stated. Available in the Kindle Store.*
