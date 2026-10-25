---
status: Running
host: "multiple nodes"
reviewed: 2026-10-25
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
Tailscale is installed on each node individually. There is no subnet router, so no node advertises the LAN to the tailnet. A service that needs the tailnet gets Tailscale inside its own container, which keeps the hypervisor off the tailnet.

The one exception, since October 2026, is the two Proxmox nodes. They are on the tailnet only so they can reach the [QDevice](qdevice.md) on the VPS. It is temporary, and they come off when a Raspberry Pi takes over that job. I had removed the first node from the tailnet earlier in October, and put it back, tagged, for this.

## Configuration
On the tailnet: the workstation, both Pi-holes, the Caddy LXC, the Wazuh manager, the VPS, the Gitea, FreshRSS, Vikunja, ntfy, Prometheus and Grafana containers, both Proxmox nodes, and my phone. An old phone and a Kali node are still listed in the admin console, both offline.

Not on the tailnet: Vaultwarden and Nextcloud.

- **Policy:** I replaced the default allow-all policy with a restricted one in `grants` syntax, with tests. Members and exit nodes reach each other as before. A `tag:pve` tag covers the two Proxmox nodes, and may reach only the VPS, on the one port the QDevice uses. Both nodes join with `--advertise-tags=tag:pve --shields-up --accept-dns=false`, and their key expiry is off. See [Tailscale tags and grants](../concepts/tailscale-tags-and-grants.md). After changing the policy I checked from my phone, with wifi off, that two tailnet-only services still loaded.
- **DNS:** there are two tailnet resolvers, in this order: the first Pi-hole and the second one, which I added in October 2026. Until then the one Pi-hole was the only resolver, and when the node it lives on went down, DNS stopped on my workstation and phone, because they use the tailnet resolver and ignore the router's DNS (see [Tailnet DNS override](../concepts/tailnet-dns-override.md) and [the incident](../incidents/dns-outage-when-the-first-node-went-down.md)). I did not add a public resolver to the list. As far as I know, Tailscale may query several nameservers at once, so some queries could skip the Pi-hole, and I have not tested that. MagicDNS is on, so tailnet device names such as the FreshRSS one resolve. MagicDNS was off earlier in the build, and I did not record when I turned it on.
- Vikunja and ntfy are served over HTTPS with `tailscale serve`. HTTPS certificates have to be enabled on the tailnet in the admin console first.
- Unprivileged LXCs need TUN passthrough in their config before Tailscale works inside them. Steps: [Enable Tailscale in an unprivileged Proxmox LXC](../runbooks/enable-tailscale-unprivileged-lxc.md).
- Key expiry is off for the FreshRSS node, the second Pi-hole and both Proxmox nodes. Turning it off for the ntfy node is on my list, and I did not record that I did it.
- I removed Tailscale from the Proxmox Kali VM in October 2026, to keep it isolated.

## Dependencies
- [Pi-hole](pi-hole.md) (two instances), as the tailnet resolvers

## Known Issues
- The old phone is still on the tailnet, offline for weeks. I have not decided whether to remove it.
- The Kali node is still listed in the admin console, offline, because I removed Tailscale from the VM without removing the node.
- A device without Tailscale cannot reach anything that only has a tailnet route, including Vaultwarden, even at home.
- A node whose key expires drops off the tailnet without any warning.

## Related Notes
- [Tailscale tags and grants](../concepts/tailscale-tags-and-grants.md)
- [Tailnet DNS override](../concepts/tailnet-dns-override.md)
- [QDevice](qdevice.md)
- [Enable Tailscale in an unprivileged Proxmox LXC](../runbooks/enable-tailscale-unprivileged-lxc.md)
- [Take a service off the public internet](../runbooks/take-a-service-off-the-public-internet.md)
- [ntfy](ntfy.md)

## Tags
#service
