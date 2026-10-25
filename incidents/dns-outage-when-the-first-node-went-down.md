---
date: 2026-10-24
status: Resolved
tags:
  - incident
  - "#dns"
---

# DNS outage when the first node went down

## What Happened
I shut down the first Proxmox node to pull a spare Optane module for the second node. While it was off, my wired desktop and my phone on wifi both lost internet access. When the node came back up, everything recovered by itself. The outage lasted as long as the node was down.

Before the shutdown I had set the router's DNS to two public resolvers in the ISP app, so that DNS would survive the Pi-hole going down. It made no difference.

## Investigation
- My first guess was that the router needed a restart for the DNS change to apply. That was wrong, since I had saved the change and the outage still followed the node.
- `resolvectl status` failed with `Failed to connect to service /run/systemd/resolve/io.systemd.Resolve`. This workstation does not use systemd-resolved.
- `/etc/resolv.conf` on the workstation is managed by Tailscale.
- `tailscale dns status` showed a single nameserver for the tailnet: the Pi-hole.
- A lookup of a name that only exists as a Pi-hole local record returned the local answer. If the router's public resolvers had been answering, that could not have happened. The Pi-hole query log showed the request, and the client address was my workstation's tailnet address.

## Root Cause
The Pi-hole is the only resolver Tailscale hands out, and it is an LXC on the node I shut down. My devices use Tailscale's DNS settings, so they never look at the router's DNS servers. With the node off, the only nameserver they knew about was gone and nothing else was configured. The router setting was never in the path.

This is the same family of problem as [the Proxmox host DNS root cause](proxmox-dns-root-cause.md): Tailscale taking over resolution on a device, then my assumptions about which resolver is in use being wrong. See [Tailnet DNS override](../concepts/tailnet-dns-override.md).

## Fix
- Left the router on public resolvers. It does no harm for devices that do not use Tailscale DNS.
- Built a second Pi-hole on the second node from a Teleporter export of the first. See [Build a second Pi-hole and fail over to it](../runbooks/build-a-second-pi-hole-and-fail-over.md).
- Added the second Pi-hole's tailnet address as the second nameserver in the tailnet DNS settings.
- Set both Proxmox nodes to resolve through the two Pi-holes, then a public resolver, with a short timeout. See [Set up DNS fail-safes on a Proxmox node](../runbooks/set-up-dns-fail-safes-on-a-proxmox-node.md).
- Tested it by stopping the first Pi-hole for under a minute. The first node still resolved, in 9.2 seconds before the timeout option and 3.0 after. An uncached name on the workstation resolved in about 0.1 seconds.
- I did not add public resolvers to the tailnet list. As far as I know, Tailscale may query several nameservers at once, so some queries could skip the Pi-hole. I have not tested that. If both Pi-holes are ever down together, `tailscale set --accept-dns=false` on the workstation makes it use the normal resolver again.

Still open from this: the two Pi-holes can drift apart, and the second Pi-hole's web UI login fails. Both are in [Pi-hole](../services/pi-hole.md).

I did not note the exact time the outage started and ended.

## Related Notes
- [Tailnet DNS override](../concepts/tailnet-dns-override.md)
- [Proxmox host DNS root cause](proxmox-dns-root-cause.md)
- [Pi-hole](../services/pi-hole.md)
- [Tailscale](../services/tailscale.md)
