---
status: Running
host: "two LXCs, one per Proxmox node"
reviewed: 2026-10-25
tags: [service]
---

# Pi-hole

## Overview
| Field | Value |
|-------|-------|
| **Purpose** | Network-wide DNS and ad-blocking, also used for local DNS records for internal-only services |
| **Host** | Two LXCs: `pihole` on the first Proxmox node (Ubuntu 22.04) and `pihole2` on the second (Debian 13, added October 2026). See [Infrastructure](../infrastructure.md). |
| **Status** | Running |

## Dependencies
- Heavily depended upon: most other hosts use it as their DNS resolver. This has caused real incidents when a host ended up pointed elsewhere, see Known Issues.
- Both Proxmox nodes and the tailnet use both instances, in order, so losing one node no longer takes DNS down. Steps: [Build a second Pi-hole and fail over to it](../runbooks/build-a-second-pi-hole-and-fail-over.md) and [Set up DNS fail-safes on a Proxmox node](../runbooks/set-up-dns-fail-safes-on-a-proxmox-node.md).

## Known Issues
- **The two instances are independent copies.** `pihole2` was built from a Teleporter export of the first. Teleporter is a one-time copy, not a sync, so a record or list change on one does not appear on the other. I have not picked a sync tool or a habit. Open.
- **`pihole2` web UI login fails** with either password I know. Resolution itself works. Open.
- **`pihole2` has no Wazuh agent and no backup yet.** Open.
- **The first instance is still on Ubuntu 22.04.** I want to rebuild it on Debian 13 from a Teleporter export once the second one has proven itself.
- **DNS single point of failure, fixed**: when the first node went down, DNS stopped on my workstation and phone, because the tailnet's only resolver was the Pi-hole on that node. See [the incident](../incidents/dns-outage-when-the-first-node-went-down.md).
- **Recurring DNS pattern, root cause found and fixed**: new LXCs created through Proxmox's web UI default to the host's DNS, which was resolving through the tailnet rather than Pi-hole. This broke a git mirror sync, a retired game server, and a Tailscale join at different points, three separate incidents, before the actual cause (Tailscale's DNS override setting silently overwriting the host's own resolver) was tracked down and fixed at the source rather than patched reactively each time. See [the incident writeup](../incidents/proxmox-dns-root-cause.md).
- **Stale answers after adding a record**: a new local record can still return the old upstream answer from cache (`EDE: 3 (Stale Answer)` in `dig` output), including a parking page from the public zone's wildcard. Restarting `pihole-FTL` clears it. Seen again when a single `dig` briefly returned two answers for one name, never proven to be anything but cache.
- Local records are only as current as the hosts behind them. A workstation on DHCP drifted to a new address and its record went stale. A DHCP reservation fixes that.

## Related Notes
- [Infrastructure](../infrastructure.md)
- [Proxmox host DNS root cause](../incidents/proxmox-dns-root-cause.md)
- [DNS outage when the first node went down](../incidents/dns-outage-when-the-first-node-went-down.md)
- [Tailnet DNS override](../concepts/tailnet-dns-override.md)

## Tags
#service
