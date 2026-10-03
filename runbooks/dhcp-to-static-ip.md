---
tags:
  - runbook
  - "#proxmox"
---

# Convert a Proxmox LXC from DHCP to static IP

Based on converting several LXCs in this lab from DHCP to static addressing. See [Proxmox](../services/proxmox.md) for host-level context.

## Step 1: Find the currently-leased address
Don't guess a new IP, use the one already assigned, so nothing else that might reference it (bookmarks, firewall rules, other configs) silently breaks:
```
pct exec <vmid> -- ip -4 addr show eth0
```
Note the `inet` address shown (with `dynamic` in the flags, confirming it's DHCP-assigned).

## Step 2: Get the current MAC address
Needed so the static config doesn't change the interface's identity:
```
pct config <vmid> | grep net0
```
Note the `hwaddr=` value.

## Step 3: Apply the static config
```
pct set <vmid> -net0 name=eth0,bridge=vmbr0,hwaddr=<MAC>,ip=<IP>/24,gw=<gateway>,type=veth
```
Add `,firewall=1` if this LXC should participate in the Proxmox firewall zone, matching whatever the other static LXCs on this host use.

## Step 4: Expect (and ignore) the hotplug error
This command usually fails partway with something like:
```
Error: ipv4: Address already assigned.
command 'lxc-attach ... ip -4 addr add ...' failed: exit code 2
```
This is only the *live hotplug* step failing, since the interface already holds the old DHCP address and the kernel refuses to add the same IP twice. The config itself still gets written to disk. Confirm with:
```
pct config <vmid> | grep net0
```
If it shows the static config you just set, the write succeeded despite the error.

## Step 5: Reboot to apply cleanly
```
pct reboot <vmid>
pct exec <vmid> -- ip -4 addr show eth0
```
Confirm the address now shows `valid_lft forever` and no `dynamic` flag.

## Notes / gotchas
- Don't skip Step 1 and pick an arbitrary "free-looking" IP, DHCP pool boundaries aren't always obvious from outside the router, and reusing the already-leased address avoids any collision risk entirely.
- The Step 4 error is expected and doesn't mean the change failed, don't retry the command, just check the config and reboot.

## Related Notes
- [Proxmox](../services/proxmox.md)
