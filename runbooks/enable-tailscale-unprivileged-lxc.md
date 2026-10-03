---
tags:
  - runbook
  - "#proxmox"
  - "#tailscale"
---

# Enable Tailscale in an unprivileged Proxmox LXC

Based on joining a host to the tailnet in this lab, full incident detail (including a real config-file bug hit along the way) in [Prometheus tailscale join](../incidents/prometheus-tailscale-join.md).

## Step 1: Check whether the TUN device is already available
```
pct exec <vmid> -- ls -la /dev/net/tun
```
If this shows the device, skip to Step 4, Tailscale can be installed directly. If it errors "No such file or directory," continue.

## Step 2: Confirm the host itself has it, and check privilege mode
```
ls -la /dev/net/tun                          # on the Proxmox host
pct config <vmid> | grep -i unprivileged
```
If the host has the device but the container doesn't, and `unprivileged: 1` is set, Proxmox's default confinement for unprivileged containers is blocking it. This is the common case. See [unprivileged LXC confinement](../concepts/unprivileged-lxc-confinement.md) for why.

## Step 3: Grant device access, edit the live config section carefully
Add these lines directly to `/etc/pve/lxc/<vmid>.conf`:
```
lxc.cgroup2.devices.allow: c 10:200 rwm
lxc.mount.entry: /dev/net/tun dev/net/tun none bind,create=file
lxc.apparmor.profile: unconfined
```

**Critical gotcha**: Proxmox stores snapshot history in the *same* `.conf` file as the live config, as `[bracket-named]` sections appended below it. A blind `echo "..." >> /etc/pve/lxc/<vmid>.conf` appends to the *end of the file*, which lands inside the last snapshot section rather than the live config if the container has any snapshots. Proxmox then reads those lines as snapshot metadata, not active config, and they silently have no effect. **Always check the file's structure first**:
```
cat /etc/pve/lxc/<vmid>.conf
```
If there are any `[snapshot-name]` sections, insert the three lines into the live section at the *top* of the file (before the first `[` line), not at the end.

Try `nesting=1` alone first (`pct set <vmid> -features nesting=1`) before reaching for the more invasive `lxc.apparmor.profile: unconfined`, it may be sufficient on its own. If it isn't, add the apparmor line too, note that once `unconfined` is set, an explicit `nesting=1` feature becomes redundant, Proxmox will say so in a warning on reboot, harmless to leave as-is.

## Step 4: Reboot and confirm
```
pct reboot <vmid>
pct exec <vmid> -- ls -la /dev/net/tun
```
Should now show the device with no error.

## Step 5: Install and connect
```
pct exec <vmid> -- bash -c "curl -fsSL https://tailscale.com/install.sh | sh"
pct exec <vmid> -- tailscale up
```
This prints a login URL, open it in a browser to approve the device.

**If `curl` fails to resolve `tailscale.com`**: check `/etc/resolv.conf` inside the container. This lab had a recurring pattern of new LXCs inheriting the wrong DNS server at creation time, root cause and fix documented in [the Proxmox DNS incident](../incidents/proxmox-dns-root-cause.md).

## Step 6: Get the tailnet IP for documentation
```
pct exec <vmid> -- tailscale ip -4
```

## Notes / gotchas
- The `.conf` snapshot-append trap in Step 3 is a general Proxmox gotcha, not specific to Tailscale, any time an `echo >> /etc/pve/lxc/<vmid>.conf` looks like it didn't take effect on a container with snapshots, check the file structure before assuming the command itself failed.
- Try the least invasive fix (`nesting=1`) before the more invasive one (`apparmor.profile: unconfined`), both may not be necessary.

## Related Notes
- [Proxmox](../services/proxmox.md)
- [Prometheus tailscale join](../incidents/prometheus-tailscale-join.md)
- [Proxmox DNS root cause](../incidents/proxmox-dns-root-cause.md)
