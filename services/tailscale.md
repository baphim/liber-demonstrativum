---
status: Running
host: "multiple nodes"
reviewed: 2026-10-20
tags: [service]
---

# Tailscale

## Overview
| Field | Value |
|-------|-------|
| **Purpose** | Private network between my devices and the lab. Most lab services are reachable only over the tailnet. |
| **Host** | Every node that needs it, see below |
| **Status** | Running |

## Access
Managed from the Tailscale admin console. Nothing is exposed to the public internet by Tailscale itself. The one public route goes through the VPS, see [Caddy](caddy.md).

## Deployment
Tailscale is installed on each node individually. There is no subnet router, so no node advertises the LAN to the tailnet. A service that needs the tailnet gets Tailscale inside its own container, which keeps the hypervisor off the tailnet. I removed the Proxmox host from the tailnet in October 2026 for that reason, and I reach it through the workstation.

## Configuration
On the tailnet: the workstation, the Pi-hole, the Caddy LXC, the Wazuh manager, the VPS, the Gitea, FreshRSS, Vikunja, ntfy, Prometheus and Grafana containers, and my phone. An old phone and a Kali node are still listed in the admin console, both offline.

Not on the tailnet: the Proxmox host, Vaultwarden and Nextcloud.

- Vikunja and ntfy are served over HTTPS with `tailscale serve`. HTTPS certificates have to be enabled on the tailnet in the admin console first.
- The Pi-hole is the tailnet resolver, with MagicDNS on, so tailnet device names such as the FreshRSS one resolve. MagicDNS was off earlier in the build, and I did not record when I turned it on. See [Pi-hole](pi-hole.md).
- Unprivileged LXCs need TUN passthrough in their config before Tailscale works inside them. Steps: [Enable Tailscale in an unprivileged Proxmox LXC](../runbooks/enable-tailscale-unprivileged-lxc.md).
- Key expiry is off for the FreshRSS node. Turning it off for the ntfy node is on my list, and I did not record that I did it.
- I removed Tailscale from the Proxmox Kali VM in October 2026, to keep it isolated.

## Dependencies
- [Pi-hole](pi-hole.md), as the tailnet resolver

## Known Issues
- The old phone is still on the tailnet, offline for weeks. I have not decided whether to remove it.
- The Kali node is still listed in the admin console, offline, because I removed Tailscale from the VM without removing the node.
- A device without Tailscale cannot reach anything that only has a tailnet route, including Vaultwarden, even at home.
- A node whose key expires drops off the tailnet without any warning.

## Related Notes
- [Enable Tailscale in an unprivileged Proxmox LXC](../runbooks/enable-tailscale-unprivileged-lxc.md)
- [Take a service off the public internet](../runbooks/take-a-service-off-the-public-internet.md)
- [ntfy](ntfy.md)

## Tags
#service
