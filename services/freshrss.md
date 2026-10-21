---
status: Running
host: "rss LXC"
tags: [service]
---

# FreshRSS

## Overview
| Field | Value |
|-------|-------|
| **Purpose** | Self-hosted RSS reader for security news feeds, with API sync to an Android reader |
| **Host** | Dedicated unprivileged LXC, see [Infrastructure](../infrastructure.md) |
| **Status** | Running |

## Access
Reachable on the LAN by a local DNS name, and off the LAN over the tailnet only. It is not proxied through [Caddy](caddy.md) and has no public route. The Google Reader API endpoint is `/api/greader.php`. The reader app uses the API password, which is separate from the web login password.

## Deployment
Native install of FreshRSS 1.30.1, no Docker. Apache with PHP 8.4 and a SQLite database, installed with the CLI tools instead of the web installer. One user, API enabled.

Full steps: [Build FreshRSS in a Proxmox LXC](../runbooks/build-freshrss-in-lxc.md).

## Configuration
- Apache vhost with `AllowEncodedSlashes On` (needed for the API) and the default site disabled.
- Feed refresh: cron every 20 minutes as `www-data`.
- OPML export: weekly script, 90 days of exports kept.
- Feeds: 15 security feeds in three folders (threat intel and advisories, news, commentary and newsletters), loaded from an OPML file.
- Android reader: FeedMe, syncing over the tailnet on plain http. The traffic never leaves the tailnet, so HTTPS in front was not needed.
- Backups: its own nightly vzdump job, separate from the weekly job that covers the rest of the lab.

## Dependencies
- [Pi-hole](pi-hole.md) for the local DNS record.
- Tailscale, running inside the container, for off-LAN access.
- [Wazuh](wazuh.md) agent, held at the manager's version.

## Known Issues
- Backups and OPML exports share a ZFS pool with the container. A pool failure would take all three. An off-pool copy is planned, destination not decided.
- Four feeds could not be confirmed by fetch before import, because the sites block bots. All four had articles after import. One of them uses a feed URL from a third-party page, so it is the first place to look if it breaks.
- The Wazuh agent is held at the manager's version with `apt-mark hold`. Unhold it before upgrading the manager.
- The local DNS name does not resolve off the LAN. Use the tailnet name.
- Pi-hole can serve a stale cached answer right after a record is added. Restarting `pihole-FTL` clears it.
- Perl locale warnings appear on every command in the container. Harmless.

## Related Notes
- [Build FreshRSS in a Proxmox LXC](../runbooks/build-freshrss-in-lxc.md)
- [Enable Tailscale in an unprivileged LXC](../runbooks/enable-tailscale-unprivileged-lxc.md)

## Tags
#service
