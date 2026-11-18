---
tags:
  - concept
  - "#databases"
---

# WAL mode in SQLite (and why backups use `.backup`)

A SQLite database in Write-Ahead Logging mode (WAL) writes data in a way meant to avoid corruption. Instead of writing straight to the database file, it writes to a secondary write-ahead log (`-wal`) file next to the main one. The main file always holds the last stable state, while recent changes sit in the secondary file until they're folded back into the main database file in a process called checkpointing. There's also a third file (`-shm`) that helps coordinate multiple processes reading and writing at once.

A plain copy command can cause problems here: you might capture the main file and the write-ahead log at two different instants, producing a torn, inconsistent snapshot instead of one coherent state. Not corruption exactly, but an incomplete or mismatched copy.

SQLite has its own online backup command for this (`sqlite3 db.sqlite3 ".backup 'dest'"`). It captures the main file and folds in whatever's in the `-wal` file as one atomic operation, producing a complete, self-contained backup with nothing left pending.

Because of that, restoring any SQLite backup means deleting any leftover `-wal` or `-shm` files first. A leftover `-wal` file belongs to whatever database it previously sat next to. If SQLite finds one next to the restored file, it will try to replay those old changes into it, corrupting the freshly-restored database instead of leaving it alone.

## Why this matters here
This is exactly the mechanism behind [Vaultwarden](../services/vaultwarden.md)'s backup script (`.backup` rather than `cp`) and why [the restore procedure](../runbooks/vaultwarden-backup-restore.md) explicitly calls for deleting any leftover `-shm`/`-wal` files as a restore step, not an optional cleanup.

## Related Notes
- [Vaultwarden](../services/vaultwarden.md)
- [Vaultwarden backup restore procedure](../runbooks/vaultwarden-backup-restore.md)

## Tags
#concept
