---
tags:
  - runbook
  - "#vaultwarden"
---

# Vaultwarden backup restore procedure

Full round-trip tested and confirmed working. See [Vaultwarden](../services/vaultwarden.md) for the backup creation side of this flow.

## Overview
Backups are encrypted tarballs containing the database, config, RSA key, and attachments. They exist in three places: the source host (last 7 days), a pulled copy on a separate workstation, and a private git repo with full history.

The decryption key is not in any of those three locations. It's stored offline, on redundant physical media, with no copy ever living on a networked host.

## Step 1: Get an encrypted backup file
Pick the source based on what's actually available, workstation copy, git repo (clone fresh if needed, older backups beyond the working tree are recoverable from git history with `git log` and `git show`), or the source host's own local copies as a last resort.

## Step 2: Get the private key
Retrieve the offline key from physical storage. Confirm the key file is present before proceeding.

## Step 3: Decrypt
```
age -d -i <path-to-key> \
    -o restored-backup.tar \
    vaultwarden-backup-<timestamp>.tar.age
```
No output on success, `age` is silent unless there's an error.

## Step 4: Extract
```
mkdir restored-data
tar -xf restored-backup.tar -C restored-data
ls -la restored-data/
```
Expect: `db.sqlite3`, `config.json`, `rsa_key.pem`, `attachments/`.

## Step 5: Stop the live service (if restoring in place)
```
sudo systemctl stop vaultwarden
```
If this is a full rebuild, install Vaultwarden fresh first, matching the original convention for binary path, working directory, and data folder.

## Step 6: Restore the files
```
sudo cp restored-data/db.sqlite3 /opt/vaultwarden/data/
sudo cp restored-data/config.json /opt/vaultwarden/data/
sudo cp restored-data/rsa_key.pem /opt/vaultwarden/data/
sudo cp -r restored-data/attachments /opt/vaultwarden/data/
sudo chown -R root:root /opt/vaultwarden/data/
```
Remove any leftover `-shm`/`-wal` files from the previous live install, the restored `db.sqlite3` is already a clean, checkpointed snapshot and doesn't need or match old WAL files:
```
sudo rm -f /opt/vaultwarden/data/db.sqlite3-shm
sudo rm -f /opt/vaultwarden/data/db.sqlite3-wal
```

## Step 7: Start and verify
```
sudo systemctl start vaultwarden
sudo systemctl status vaultwarden
sudo journalctl -u vaultwarden -n 50 --no-pager
```
Log into the web vault, confirm:
- Login works for all accounts
- Vault items present and correct as of backup date
- Attachments open correctly
- 2FA still works (depends on `rsa_key.pem` being correctly restored, it's used for session signing)

## Step 8: Clean up
```
rm -f restored-backup.tar
rm -rf restored-data/
```
Securely wipe any copy of the private key made outside its normal storage, and return physical media to secure storage.

## Notes / gotchas
- **WAL mode**: the live install uses `db.sqlite3` with `-shm`/`-wal` files. Backups are taken via `sqlite3 .backup` for a consistent snapshot, so the restored `db.sqlite3` is already clean, never copy old `-shm`/`-wal` files alongside it. See [WAL mode in SQLite](../concepts/wal-mode-sqlite.md).
- **`rsa_key.pem` matters**: losing or mismatching it breaks existing session tokens and 2FA state. Always restore it alongside the same backup's database, never mix a database from one backup with an RSA key from another.
- **The decryption key is what makes every backup usable.** It exists on redundant offline media, real redundancy, but nowhere networked. All copies would need to be lost for backups to become unrecoverable.
- **Full host loss**: also review the systemd unit for environment variables and paths, since these aren't part of the backed-up data path and would need to be recreated separately.

## Related Notes
- [Vaultwarden](../services/vaultwarden.md)
