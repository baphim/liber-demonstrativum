---
status: Running
host: "vaultwarden LXC"
tags: [service]
---

# Vaultwarden

## Overview
| Field | Value |
|-------|-------|
| **Purpose** | Self-hosted password manager (Bitwarden-compatible) |
| **Host** | vaultwarden LXC, see [Infrastructure](../infrastructure.md) |
| **Status** | Running |

## Access
| Field | Value |
|-------|-------|
| **URL** | `vaultwarden.baphim.xyz`, proxied by [Caddy](caddy.md) to `<lan-ip>:8080` |
| **Port** | 8080 (backend). Tailnet only. Local DNS resolves the name to the Caddy container's tailnet address, and there is no public DNS record or public proxy block. Still served with a real TLS cert (Let's Encrypt via DNS-01), which does not need the service to be reachable from the internet. |

It was internet-facing through a VPS until October 2026. Every device I use it from is already on the tailnet, so the public route was removed. See [Take a service off the public internet](../runbooks/take-a-service-off-the-public-internet.md).

## Deployment
- Built from source, not installed via package manager or Docker.
- Binary deployed to `/opt/vaultwarden/vaultwarden`, run as a systemd service.
- Web vault unpacked separately from `bw_web_builds` releases.
- Database: sqlite.

## Configuration
- Update steps: [Update Vaultwarden built from source](../runbooks/update-vaultwarden-from-source.md).
- Rebuilding requires the Rust/Cargo toolchain on the host. It was removed during a storage cleanup at one point and had to be reinstalled before the next update could proceed, now tracked as a known dependency rather than a surprise.
- A build needs the LXC's memory bumped to 4096MB (`cargo build --release` is memory-hungry).
- **fail2ban**: a custom filter matches failed-login messages from the systemd journal (Vaultwarden logs to journald, no file redirect). Jail: 5 attempts in 600 seconds triggers a 1-hour ban. Verified against real historical failed-login attempts before going live.
- A systemd unit typo (`Environment-DATA_FOLDER` instead of `Environment=DATA_FOLDER`) was silently ignored for some time, Vaultwarden was finding its data directory by coincidence, not because the setting was taking effect. Found and fixed during a routine check.

## Backup
Runs daily via a systemd timer, executing a backup script. Process:
1. Takes a consistent sqlite snapshot via `.backup` (WAL-safe, avoids a torn copy of the live database).
2. Copies the config, the RSA key (needed for session/2FA continuity on restore), and attachments.
3. Tars the staged files, encrypts the tarball to an `age` public key, then deletes the unencrypted intermediate.
4. Keeps the 7 most recent encrypted backups locally, pruning older ones. Nothing leaves the host except already-encrypted data.
5. Reports success/failure and a timestamp to Prometheus via node_exporter's textfile collector, feeding a Grafana dashboard.

Backups exist in three places: the source host, a pull to a separate workstation over a restricted SSH account (own key, read access scoped to only the encrypted backup files), and a private git repo with full history. The decryption key is stored offline on redundant physical media, never on any networked host.

Full restore procedure, tested and verified: [Vaultwarden backup restore procedure](../runbooks/vaultwarden-backup-restore.md).

## Dependencies
- Reachable through [Caddy](caddy.md)'s reverse proxy.
- Monitored by [Wazuh](wazuh.md).
- A dedicated, restricted SSH account is required for the daily backup pull. If that account or its key is ever removed, the pull stops silently, only a local desktop notification on failure would catch it.

## Known Issues
- Hardening pass complete: admin token set and hashed (not plaintext), new signups disabled, outbound mail configured so security emails actually go out, fail2ban installed and verified, 2FA enabled on the account.
- Rust/Cargo must stay installed for future updates to be buildable. Removing it to save disk space creates a silent trap the next time an update is needed.
- A device without Tailscale cannot reach Vaultwarden anywhere, including at home. The Bitwarden clients keep an offline vault cache, so passwords can still be read in a pinch.
- Reachable only through [Caddy](caddy.md), so a Caddy outage takes it down for every client. That happened once when an `apt upgrade` broke Caddy, see [the incident](../incidents/caddy-apt-upgrade-removed-dns-module.md). Clients keep their offline vault cache, so passwords were still readable, but edits failed.
- No automated tracking of new releases yet. Updates have so far happened reactively, after noticing client-side errors from an outdated web vault build.

## Related Notes
- [Infrastructure](../infrastructure.md)
- [Vaultwarden backup restore procedure](../runbooks/vaultwarden-backup-restore.md)
- [Take a service off the public internet](../runbooks/take-a-service-off-the-public-internet.md)
- [Update Vaultwarden built from source](../runbooks/update-vaultwarden-from-source.md)
- [Caddy apt upgrade removed the DNS provider module](../incidents/caddy-apt-upgrade-removed-dns-module.md)

## Tags
#service
