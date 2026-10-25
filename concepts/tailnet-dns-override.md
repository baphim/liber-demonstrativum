# Tailnet DNS override

## What It Is
When a device has Tailscale's DNS setting on (`accept-dns`), Tailscale takes over how that device resolves names. The nameservers I list in the admin console are used instead of whatever the router handed out over DHCP. My router's DNS settings do not apply to those devices at all.

## How It Works
- With `accept-dns` on, Tailscale rewrites `/etc/resolv.conf` (the file says not to edit it by hand) and sends lookups to its own resolver, which forwards them to the tailnet's configured nameservers. MagicDNS names resolve this way too.
- If the admin console has one nameserver and it goes down, the device has nowhere else to ask. Nothing falls back to the router.
- Turning it off on a device (`--accept-dns=false`, or `tailscale set --accept-dns=false`) puts that device back on normal DHCP or static DNS.
- Proxmox LXCs take their default DNS from the host at creation, so a host with `accept-dns` on can pass the wrong resolver to every new container. That was the earlier problem in [the Proxmox DNS root cause](../incidents/proxmox-dns-root-cause.md).
- A nameserver in the tailnet list has to be a tailnet address, not a LAN one. A LAN address only works for devices on the LAN, and I have no subnet router, so my phone away from home could not reach it.

## In My Setup
For a while the Pi-hole was the only nameserver in my tailnet DNS settings, and it is an LXC on the first Proxmox node. When I shut that node down, my workstation and phone lost name resolution, even though I had changed the router's DNS to public resolvers beforehand. The router setting was never being used. See [the incident](../incidents/dns-outage-when-the-first-node-went-down.md).

How I tell which resolver is in use: look at `/etc/resolv.conf`, run `tailscale dns status`, and check which client address shows up in the Pi-hole query log. A tailnet address as the client means the query came through Tailscale.

The fix was a second Pi-hole on the second node, added as the second tailnet nameserver. The tailnet list is the two Pi-holes and nothing else. I left public resolvers out of it, because as far as I know Tailscale may query several nameservers at once, so some lookups could skip the Pi-hole. I have not tested that.

Both Proxmox nodes run with `--accept-dns=false`, so they resolve through their own settings and not the tailnet's. Steps: [Set up DNS fail-safes on a Proxmox node](../runbooks/set-up-dns-fail-safes-on-a-proxmox-node.md).

## Related Notes
- [Pi-hole](../services/pi-hole.md)
- [Tailscale](../services/tailscale.md)
- [Tailscale tags and grants](tailscale-tags-and-grants.md)
- [Proxmox host DNS root cause](../incidents/proxmox-dns-root-cause.md)
- [DNS outage when the first node went down](../incidents/dns-outage-when-the-first-node-went-down.md)
