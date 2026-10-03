---
date: 2026-09-19
status: Resolved
tags:
  - incident
  - "#prometheus"
  - "#tailscale"
---

# Prometheus tailscale join, TUN device and lxc.conf snapshot bug

## What Happened
Decided to join the Prometheus host to the tailnet, it was the one core host left off it. Installing Tailscale needs `/dev/net/tun`, which wasn't present in the container. Three separate problems had to be worked through before it was.

## Investigation
- The device was missing inside the LXC, but present on the Proxmox host itself, so the host side was fine, the container just wasn't getting access to it.
- Confirmed the container is unprivileged, which is why Proxmox's default confinement was blocking the device.
- First attempt: enabling the `nesting` feature and rebooting. Didn't fix it.
- Second attempt: appending the needed cgroup and AppArmor config lines directly to the container's config file. Still didn't fix it, the device kept coming back missing after every reboot.
- Checked the actual config file contents rather than continuing to guess, and found the real problem there.

## Root Cause
Proxmox stores snapshot history in the *same* config file as the live config, as later bracket-named sections appended below it. Every blind append to the file had been landing inside the last snapshot section rather than the live config section at the top. Proxmox was reading those lines as snapshot metadata, not as active container config, so none of the intended changes were actually taking effect despite looking like they'd been applied.

Side finding while reading through the snapshot history: an old snapshot's comment noted an unexpected error setting up a metrics target on another host months earlier. That's likely why that host still had no monitoring coverage, something broke during the original setup and was apparently never revisited, worth following up on separately. (It was, see [Install node_exporter](../runbooks/install-node-exporter.md).)

A fourth, unrelated problem showed up once the device was finally present: DNS resolution failed entirely, the same recurring pattern seen in two earlier incidents on other hosts. This was the third time the same failure mode had hit a different host, which is what finally prompted tracking down the actual root cause rather than patching it again. See [Proxmox host DNS root cause](proxmox-dns-root-cause.md).

## Fix
1. Removed the misplaced config lines from wherever they'd landed, then re-inserted them in the live config section where Proxmox actually reads them.
2. Rebooted, confirmed the device was finally present.
3. Fixed DNS the same way as the prior incidents, pointed the resolver at the correct internal DNS server and made it persistent.
4. Installed Tailscale and connected, approved the device in browser.

## Related Notes
- [Prometheus](../services/prometheus.md)
- [Proxmox host DNS root cause](proxmox-dns-root-cause.md)

## Tags
#incident
