---
tags:
  - runbook
  - "#proxmox"
  - "#dns"
---

# Set up DNS fail-safes on a Proxmox node

A Proxmox node that resolves through a Pi-hole running on itself or on its neighbor needs somewhere else to ask when that Pi-hole is down. Based on the fix for [the DNS outage](../incidents/dns-outage-when-the-first-node-went-down.md). Values in angle brackets are placeholders. Run on each node, as root.

## Step 1: Set three resolvers in order
```
pvesh set /nodes/$(hostname)/dns --search <search-domain> --dns1 <pihole-1> --dns2 <pihole-2> --dns3 1.1.1.1
```
Keep each node's existing search domain. Mine differed between nodes. The public resolver is last, so it is only reached if both Pi-holes fail to answer.

## Step 2: Shorten the timeout
```
echo 'options timeout:1 attempts:2' >> /etc/resolv.conf
```
A plain `resolv.conf` waits five seconds per nameserver before moving to the next one. This line cuts that to one.

This is a hand edit. Saving the DNS settings again through the Proxmox GUI or `pvesh` rewrites `resolv.conf` and drops the line, so I have to reapply it after any DNS change.

## Step 3: Make sure Tailscale is not in the way
On a node that is on the tailnet, `--accept-dns=false` has to be set, or Tailscale rewrites `resolv.conf` and the order above means nothing. See [Tailnet DNS override](../concepts/tailnet-dns-override.md).

## Step 4: Test by stopping the first Pi-hole
```
pct stop <pihole-id>
time getent hosts <name-the-pihole-answers>
pct start <pihole-id>
```
Do it for under a minute and check `pct status <pihole-id>` after.

My numbers: on the node that had to fall back to the second Pi-hole, the lookup took 9.2 seconds before the timeout option and 3.0 after, and it resolved both times. On the workstation, an uncached name (a random one under `127.0.0.1.nip.io`) took about 0.1 seconds. My first workstation test used a plain name that could have been cached, so I do not count it.

## Router DNS
I left the router on public resolvers. Devices that are not on the tailnet never use the Pi-holes, so this has no effect on them either way, and it is not part of the fail-safe.

## Related Notes
- [Build a second Pi-hole and fail over to it](build-a-second-pi-hole-and-fail-over.md)
- [Proxmox host DNS root cause](../incidents/proxmox-dns-root-cause.md)
- [Pi-hole](../services/pi-hole.md)
