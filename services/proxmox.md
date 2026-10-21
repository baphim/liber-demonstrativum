---
status: Running
host: "Bare metal"
reviewed: 2026-10-20
tags: [service]
---

# Proxmox

## Overview
| Field | Value |
|-------|-------|
| **Purpose** | Hypervisor for the whole lab, hosts every LXC and VM. See [Infrastructure](../infrastructure.md). |
| **Host** | Bare metal |
| **Status** | Running |

## Configuration
- Weekly backup job: zstd compression, snapshot mode, keeps the last 3 backups per guest. Targets every persistent LXC and the Wazuh VM, deliberately excludes the pentest targets, which are on-demand rather than persistent state worth preserving.
- A separate checker script, on its own timer shortly after the backup job, reports backup success/failure to Prometheus by checking Proxmox's own task log rather than hooking into the backup job directly. Full detail in [Prometheus](prometheus.md).

- The host is not on the tailnet (I removed it in October 2026). SSH is key-only, and I reach it through the workstation.

## Dependencies
- Nothing upstream, this is the base layer everything else runs on.
- Depended on by every other machine and service in the lab.

## Known Issues
- **DNS root cause found and fixed**: new LXCs were repeatedly inheriting the wrong DNS server at creation time, traced back to Tailscale's `accept-dns` setting silently overwriting the host's own resolver, which new containers then inherit by default. Fixed at the source rather than patched reactively each time it recurred. Full writeup in [the incident](../incidents/proxmox-dns-root-cause.md).
- New LXCs created through the web UI have more than once ended up without a default gateway set, requiring a manual fix after the fact.
- A separate, WAN-only network bridge is already configured for a planned media-automation stack, intentionally not yet built, waiting on a second Proxmox node. Not a gap, just infrastructure that exists ahead of the service it's meant for.

## Related Notes
- [Prometheus](prometheus.md)
- [Proxmox host DNS root cause](../incidents/proxmox-dns-root-cause.md)
- [Convert a Proxmox LXC from DHCP to static IP](../runbooks/dhcp-to-static-ip.md)
- [Enable Tailscale in an unprivileged Proxmox LXC](../runbooks/enable-tailscale-unprivileged-lxc.md)

## Tags
#service
