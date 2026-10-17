---
status: Running
host: "[Infrastructure](../infrastructure.md)"
tags: [service]
---

# Nextcloud

## Overview
| Field | Value |
|-------|-------|
| **Purpose** | Personal file storage |
| **Host** | Dedicated LXC, see [Infrastructure](../infrastructure.md) |
| **Status** | Running |

## Access
`nextcloud.baphim.xyz`, proxied by [Caddy](caddy.md). Real Let's Encrypt cert via DNS-01, same pattern as [Vaultwarden](vaultwarden.md).

## Deployment
Manual install on a Debian LXC with the web root under `/var/www/nextcloud` and the data directory outside the web root. I built it by following a walkthrough and did not write down how, which cost time months later when it needed updating. The build is now reconstructed in the update runbook.

## Configuration
- Maintenance goes through `occ`, run as the web server user, never root.
- Updates are done from the command line: [Update Nextcloud from the command line](../runbooks/update-nextcloud-from-the-cli.md).
- Always take a `vzdump` of the container before an update.

## Dependencies
- [Caddy](caddy.md) for TLS and the public name.

## Known Issues
- The web updater did not work when the install was several releases behind. I never found the root cause, and updated from the command line instead. Between 33.0.3 and 34.0.3 the updater's download step also failed twice on the major upgrade, which I worked around with a manual, signature-verified download.
- No automated tracking of new releases. Updates are reactive.

## Related Notes
- [Infrastructure](../infrastructure.md)
- [Caddy](caddy.md)
- [Vaultwarden](vaultwarden.md)
- [Update Nextcloud from the command line](../runbooks/update-nextcloud-from-the-cli.md)

## Tags
#service
