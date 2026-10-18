---
status: Running
host: "[Infrastructure](../infrastructure.md)"
tags: [service]
---

# Vikunja

## Overview
| Field | Value |
|-------|-------|
| **Purpose** | Self-hosted task manager, replacing a hosted to-do app |
| **Host** | Dedicated unprivileged LXC, see [Infrastructure](../infrastructure.md) |
| **Status** | Running |

## Access
Tailnet only. HTTPS comes from `tailscale serve`, which proxies to the local port inside the container, so there is no [Caddy](caddy.md) entry, no local DNS record and no public route. Used from the web UI and the Vikunja Android app, with Tailscale connected on the phone.

## Deployment
Native `.deb` package (v2.7.0) in an unprivileged Debian 12 LXC with `nesting=1` and TUN passthrough for Tailscale. SQLite database. I chose the package over Docker to avoid Docker-in-LXC nesting problems, since Vikunja is a single Go binary. Steps: [Deploy Vikunja in a Debian LXC](../runbooks/deploy-vikunja-in-a-debian-lxc.md).

## Configuration
- The config file is mode 600 and holds the service secret.
- `service.publicurl` must match the tailnet HTTPS URL exactly, because it drives the CORS allowlist.
- `service.secret` replaces the deprecated `JWTSecret` key in v2. Older guides still show the old key.
- Registration is closed after the single account was created.
- Mailer not configured.
- Wazuh agent installed and pinned to the manager's version with `apt-mark hold`.
- Covered by the weekly vzdump backup job.
- Tasks were imported once from the old app's CSV backup, with a snapshot taken first and deleted afterward.

## Dependencies
- [Proxmox](proxmox.md)
- Tailscale, inside the container, for access and for HTTPS certificates
- [Wazuh](wazuh.md)

## Known Issues
- No SMTP, so no email reminders.
- Subtask checkboxes did not survive the import and were re-added by hand.
- Habit tracking, a focus timer, a calendar view and a watch app are not covered by Vikunja. Replacements are not chosen yet, so the old account stays for now.
- After switching `publicurl` to the tailnet URL, logins from the LAN address stop working. The URL typed into the Android app has to match it exactly.
- Planned: connect Claude Code to the Vikunja MCP server.

## Related Notes
- [Deploy Vikunja in a Debian LXC](../runbooks/deploy-vikunja-in-a-debian-lxc.md)
- [Enable Tailscale in an unprivileged Proxmox LXC](../runbooks/enable-tailscale-unprivileged-lxc.md)
- [Wazuh](wazuh.md)

## Tags
#service
