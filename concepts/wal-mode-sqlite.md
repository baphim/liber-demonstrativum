---
tags:
  - concept
  - "#databases"
---

# WAL mode in SQLite (and why backups use `.backup`)

A SQLite database in Write-Ahead Logging mode (WAL) writes data in a way as to prevent database corruption. Instead of writing data directly to the database file, it writes it to a secondary write-ahead log (`-wal`) file stored next to the main file. The main file exists at any point as the last stable state it was in, and recent changes are written to the secondary file. Those recent changes are regularly written into the main database file permanently in a process known as "checkpointing." Additionally, there is a tertiary file (`-shm`) that helps coordinate multiple processes accessing data simultaneously.

When creating SQLite backups with a plain copy command, this can create issues: you might capture the main file and the write-ahead log at two different instants in time, producing a torn, inconsistent snapshot rather than one coherent state, not corruption exactly, but an incomplete or mismatched copy.

To solve this, SQLite has its own built-in online backup command (`sqlite3 db.sqlite3 ".backup 'dest'"`). This command captures the main file and folds in whatever's in the `-wal` file as one single atomic operation, producing a complete, self-contained backup with nothing pending.

Due to this mechanism, when restoring any SQLite backup, it is important to delete any pre-existing `-wal` or `-shm` files before restoring. A leftover `-wal` file belongs to whatever database it sat next to previously, if SQLite finds one next to the restored file, it will try to replay those old changes into it, corrupting the freshly-restored (and already-complete) database rather than leaving it alone.

## Why this matters here
This is exactly the mechanism behind [Vaultwarden](../services/vaultwarden.md)'s backup script (`.backup` rather than `cp`) and why [the restore procedure](../runbooks/vaultwarden-backup-restore.md) explicitly calls for deleting any leftover `-shm`/`-wal` files as a restore step, not an optional cleanup.

## Related Notes
- [Vaultwarden](../services/vaultwarden.md)
- [Vaultwarden backup restore procedure](../runbooks/vaultwarden-backup-restore.md)

## Tags
#concept
