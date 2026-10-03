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

## Related Notes
- [Infrastructure](../infrastructure.md)
- [Proxmox host DNS root cause](../incidents/proxmox-dns-root-cause.md)

## Tags
#service
