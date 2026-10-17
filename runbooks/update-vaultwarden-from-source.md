---
tags:
  - runbook
  - "#vaultwarden"
---

# Update Vaultwarden built from source

Updates a Vaultwarden instance that was built from source, along with the matching web vault. I did this when browser extension logins started failing because the server and web vault were several releases behind. See [Vaultwarden](../services/vaultwarden.md).

## Step 1: Work out what is installed
If the original build was not documented, find it.
```
systemctl status vaultwarden
ps aux | grep -i vaultwarden
find / -iname "*vaultwarden*" 2>/dev/null | grep -v proc
```
Mine was built from a git checkout, with the binary deployed to `/opt/vaultwarden/vaultwarden` and run as a systemd service.

## Step 2: See how far behind it is
```
cd <source-dir>
git status
git log -1
git fetch origin
git tag --sort=-v:refname | head
```
Check out the release tag you want.

## Step 3: Make sure a backup exists
Check the backup timer and confirm a recent encrypted backup is there to restore from before touching anything. See [Vaultwarden backup restore procedure](vaultwarden-backup-restore.md).

## Step 4: Give the build enough memory
The release build is memory-hungry. From the Proxmox host:
```
pct set <ctid> -memory 4096
```
Remember to size it back down afterward if the service does not need it.

## Step 5: Check the toolchain
```
source "$HOME/.cargo/env"
rustc --version
cargo --version
```
If the toolchain is gone, reinstall it with rustup and install the build dependencies (`build-essential`, `pkg-config`, `libssl-dev`, and the SQLite development package). Mine had been removed in a storage cleanup, which is the reason this stays documented now.

## Step 6: Build
Match the feature flag to the database in use. Mine is SQLite.
```
cargo build --release --features sqlite
```

## Step 7: Swap the binary and web vault
Keep the old binary until the new one proves itself. Use the web vault release that matches the server release.
```
systemctl stop vaultwarden
cp /opt/vaultwarden/vaultwarden /opt/vaultwarden/vaultwarden.bak
cp <source-dir>/target/release/vaultwarden /opt/vaultwarden/vaultwarden
cd /tmp
wget https://github.com/dani-garcia/bw_web_builds/releases/download/v<web-version>/bw_web_v<web-version>.tar.gz
rm -rf /opt/vaultwarden/web-vault
mkdir /opt/vaultwarden/web-vault
tar -xzf /tmp/bw_web_v<web-version>.tar.gz -C /opt/vaultwarden/web-vault --strip-components=1
```

## Step 8: Start and check
```
systemctl start vaultwarden
journalctl -u vaultwarden -f
```
A clean start ends with the Rocket "has launched" line.

## Verification
- The journal is free of errors after start.
- A client logs in, including the browser extension that was failing.
- A new backup run completes.

## Lessons
- Do not remove the Rust toolchain to save space. The next update needs it.
- Updates are still reactive. There is no release tracking, so the trigger was a client error.

## Related Notes
- [Vaultwarden](../services/vaultwarden.md)
- [Vaultwarden backup restore procedure](vaultwarden-backup-restore.md)

## Tags
#runbook
