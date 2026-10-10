---
status: Running
host: "[Infrastructure](../infrastructure.md)"
tags: [service]
---

# Pi-hole

## Overview
| Field | Value |
|-------|-------|
| **Purpose** | Network-wide DNS and ad-blocking, also used for local DNS records for internal-only services |
| **Host** | [Infrastructure](../infrastructure.md) |
| **Status** | Running |

## Dependencies
- Heavily depended upon: most other hosts use it as their DNS resolver. This has caused real incidents when a host ended up pointed elsewhere, see Known Issues.

## Known Issues
- **Recurring DNS pattern, root cause found and fixed**: new LXCs created through Proxmox's web UI default to the host's DNS, which was resolving through the tailnet rather than Pi-hole. This broke a git mirror sync, a retired game server, and a Tailscale join at different points, three separate incidents, before the actual cause (Tailscale's DNS override setting silently overwriting the host's own resolver) was tracked down and fixed at the source rather than patched reactively each time. See [the incident writeup](../incidents/proxmox-dns-root-cause.md).
- **Stale answers after adding a record**: a new local record can still return the old upstream answer from cache (`EDE: 3 (Stale Answer)` in `dig` output), including a parking page from the public zone's wildcard. Restarting `pihole-FTL` clears it. Seen again when a single `dig` briefly returned two answers for one name, never proven to be anything but cache.
- Local records are only as current as the hosts behind them. A workstation on DHCP drifted to a new address and its record went stale. A DHCP reservation fixes that.

## Related Notes
- [Infrastructure](../infrastructure.md)
- [Proxmox host DNS root cause](../incidents/proxmox-dns-root-cause.md)

## Tags
#service
