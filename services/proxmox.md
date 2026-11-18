---
status: Running
host: "Bare metal, two nodes"
reviewed: 2026-10-25
tags: [service]
---

# Proxmox

## Overview
| Field | Value |
|-------|-------|
| **Purpose** | Hypervisor for the whole lab, hosts every LXC and VM. See [Infrastructure](../infrastructure.md). |
| **Host** | Bare metal, a two-node cluster since October 2026 |
| **Status** | Running |

## Configuration
- Two nodes in one cluster, with a [QDevice](qdevice.md) on a VPS as the third vote. Steps: [Join a second Proxmox node and add a QDevice](../runbooks/join-a-second-proxmox-node-and-add-a-qdevice.md). Why a third vote: [Quorum](../concepts/quorum.md).
- The cluster shares config and the web UI. Storage definitions are shared too, so each storage is restricted to the node that actually has it. Shell config and SSH logins are still per node.
- The second node boots from a small Optane module. Its guest storage is an old, worn SSD set up as LVM-thin, see [Turn a worn SSD into LVM-thin storage](../runbooks/turn-a-worn-ssd-into-lvm-thin-storage.md) and [SSD wear and SMART](../concepts/ssd-wear-and-smart.md).
- Weekly backup job on the first node: zstd compression, snapshot mode, keeps the last 3 backups per guest. Targets every persistent LXC and the Wazuh VM on that node, deliberately excludes the pentest targets, which are on-demand rather than persistent state worth preserving.
- A separate checker script, on its own timer shortly after the backup job, reports backup success/failure to Prometheus by checking Proxmox's own task log rather than hooking into the backup job directly. Full detail in [Prometheus](prometheus.md).
- Both nodes resolve DNS through the two Pi-holes, then a public resolver, with a short timeout. See [Set up DNS fail-safes on a Proxmox node](../runbooks/set-up-dns-fail-safes-on-a-proxmox-node.md).
- The nodes are on the tailnet only to reach the QDevice (I removed the first one in October 2026, then put it back, tagged, for this). SSH is key-only, and I reach them through the workstation.

## Dependencies
- Nothing upstream, this is the base layer everything else runs on.
- The cluster itself depends on the QDevice host for its third vote.
- Depended on by every other machine and service in the lab.

## Known Issues
- Second node has no backups yet. The backup job and its storage are on the first node, so guests on the second node, including the second Pi-hole, are not covered. Open.
- DNS root cause found and fixed: new LXCs were repeatedly inheriting the wrong DNS server at creation time, traced back to Tailscale's `accept-dns` setting silently overwriting the host's own resolver, which new containers then inherit by default. Fixed at the source rather than patched reactively each time it recurred. Full writeup in [the incident](../incidents/proxmox-dns-root-cause.md). A related outage in October 2026 is in [DNS outage when the first node went down](../incidents/dns-outage-when-the-first-node-went-down.md).
- Losing the QDevice host and one node together freezes the survivor, see [QDevice](qdevice.md).
- The `options timeout` line in `resolv.conf` is a hand edit, and saving DNS settings in the Proxmox GUI drops it.
- New LXCs created through the web UI have more than once ended up without a default gateway set, requiring a manual fix after the fact.
- A separate, WAN-only network bridge is already configured on the first node for a planned media-automation stack, intentionally not yet built. Not a gap, just infrastructure that exists ahead of the service it's meant for.

## Related Notes
- [QDevice](qdevice.md)
- [Quorum](../concepts/quorum.md)
- [Prometheus](prometheus.md)
- [Proxmox host DNS root cause](../incidents/proxmox-dns-root-cause.md)
- [Convert a Proxmox LXC from DHCP to static IP](../runbooks/dhcp-to-static-ip.md)
- [Enable Tailscale in an unprivileged Proxmox LXC](../runbooks/enable-tailscale-unprivileged-lxc.md)

## Tags
#service
