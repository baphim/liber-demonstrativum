---
tags:
  - runbook
  - "#pihole"
  - "#proxmox"
  - "#tailscale"
---

# Build a second Pi-hole and fail over to it

Based on building a second Pi-hole in an unprivileged LXC on my second Proxmox node, after DNS went down with the first node (see [the incident](../incidents/dns-outage-when-the-first-node-went-down.md)). Values in angle brackets are placeholders.

## Choices
- Debian 13, because I want my LXCs on one OS and 12 is the outlier now.
- Static address, 1 core, 512 MB of RAM, a 4 GB disk on the node's own storage.
- Same settings as the first Pi-hole, through Teleporter.
- It joins the tailnet so it can be a second tailnet nameserver.

## Step 1: Check the ID and get the template
`pvesh get /cluster/nextid` prints the ID in quotes. My first check never passed because of that. The ID I planned to use was already taken by another container, so the next free one became this container.

The template list matched both architectures and my command took the last one, so I downloaded the arm64 build. It will not run on this CPU. Pick the `amd64` name explicitly, and remove a wrong download with `pveam remove local:vztmpl/<name>`. The `local` storage is per node, so each node needs its own copy of a template.

## Step 2: Create the container
As one line. The first time I pasted it with line continuations, the `--rootfs` option was lost, so Proxmox fell back to `local` and failed with `storage 'local' does not support container directories`. It also left a stub `.conf` with no hostname, which I removed before trying again.
```
pct create <id> local:vztmpl/<amd64-template> --hostname <name> --cores 1 --memory 512 --swap 512 --rootfs <storage>:4 --unprivileged 1 --features nesting=1 --onboot 1 --nameserver 1.1.1.1 --net0 name=eth0,bridge=vmbr0,ip=<address>/24,gw=<gateway>
```
Add the two TUN lines from [Enable Tailscale in an unprivileged Proxmox LXC](enable-tailscale-unprivileged-lxc.md) to `/etc/pve/lxc/<id>.conf`, take a snapshot called `pre-install`, and check `pct config <id>` and `pct listsnapshot <id>`.

## Step 3: Install Pi-hole
`pct enter` gives a shell with no proper controlling terminal, and the installer's dialogs break in it.

- `curl ... | bash` exited at the static IP notice, and the first time it drew a half-finished sky blue screen. Setting `TERM=xterm-256color` did not help.
- Running the saved script got the first dialog up, then failed with `cannot open tty-output`.
- What worked was a fake terminal: `script -qfc "bash /root/basic-install.sh" /dev/null`.

It never asked about the interface, the web interface or privacy mode. As far as I can tell that is normal for v6, where there is one interface, the web server is built in, and privacy mode is a setting. `pihole status` and a `dig @127.0.0.1` for a public name confirmed it works. `pihole` was "command not found" once and fine on the next attempt. I did not record what changed.

## Step 4: Import the first Pi-hole's settings, then set the password
In the first Pi-hole, export from Settings, Teleporter. Import everything on the new one. Check that `dns.listeningMode` and the upstreams match, and that both answer for a local record.

Do not set a password before the import. I ran `pihole setpassword` first, and the import replaced it with the first Pi-hole's password. Neither the password I set nor the installer's logged me in afterward, which is the open problem below. I do not know that the two are connected.

## Step 5: Join the tailnet
```
tailscale up --accept-dns=false --hostname=<name>
```
Disable key expiry in the admin console. Then add its tailnet address as the second nameserver in the tailnet DNS settings, below the first Pi-hole. It has to be the tailnet address and not the LAN one, because I have no subnet router, so a LAN address would not work away from home. Check with `tailscale dns status`.

## Step 6: Fail-safes and a test
Follow [Set up DNS fail-safes on a Proxmox node](set-up-dns-fail-safes-on-a-proxmox-node.md).

## Open
- The web UI login on the new Pi-hole fails with either password. With the new password the page does not load and shows no error. I may restart the container to see if that clears it.
- The two Pi-holes are independent copies. A record or list change on one does not appear on the other. A Teleporter import is a one-time copy, not a sync, and I have not picked a sync tool or a habit.
- The Wazuh agent is not on the new container yet. It has to be pinned to the manager's version and held with `apt-mark hold`, like the rest.
- No backup covers this container yet.

## Related Notes
- [Pi-hole](../services/pi-hole.md)
- [Tailnet DNS override](../concepts/tailnet-dns-override.md)
- [DNS outage when the first node went down](../incidents/dns-outage-when-the-first-node-went-down.md)
