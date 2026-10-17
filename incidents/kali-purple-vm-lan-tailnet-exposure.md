---
date: 2026-10-16
status: Resolved
tags:
  - incident
  - "#libvirt"
  - "#firewall"
---

# Kali Purple VM exposed to the LAN and tailnet

## What Happened
The Kali Purple VM on my workstation could start connections to my LAN, to my tailnet, and to SSH on the workstation itself. I had been working from the assumption that libvirt's default NAT network kept it off my LAN, and that was wrong. I found it during the isolation check at the end of the build. The exposure ran from when I built the VM until I put lockdown rules in place the same evening. I did not run any scans or offensive tools from the VM in that window.

## Investigation
From the VM, the default route went through the NAT gateway. A ping and an HTTP request to a LAN device both succeeded, and a ping to a tailnet device succeeded too. Port 22 on the workstation was open from the VM, while the other host ports I probed were closed. On the workstation, firewalld is the active firewall, and its `libvirt` zone allowed `dhcp dhcpv6 dns ssh tftp` and rejected everything else.

## Root Cause
libvirt NAT only blocks inbound connections from the LAN. Outbound traffic from the VM is masqueraded through the workstation, so the VM can reach anything the workstation can reach, including the tailnet through the workstation's own Tailscale. On top of that, the `ssh` entry in the `libvirt` firewalld zone left the workstation's SSH reachable from the VM. I assumed NAT meant isolation, and the advice I was following said the same. The reasoning behind the mistake is written up in [NAT is not isolation](../concepts/nat-is-not-isolation.md).

## Fix
I chose internet only over a fully isolated network, to match the Proxmox Kali VM.

1. Removed `ssh` and `tftp` from the `libvirt` firewalld zone, runtime and permanent, without a reload.
2. Added an nftables table that rejects forwarded traffic from the libvirt bridge to the private ranges, the tailnet range and link-local. The internet stays open.
3. Made it persistent with a oneshot systemd unit.
4. Tested from the VM. The host ports were closed, the LAN and tailnet pings failed, and the internet ping and `apt update` still worked.
5. Rebooted the workstation and confirmed the lockdown was still in place.

The rule applies to every VM on the `default` network, so future VMs there are covered too. Steps: [Lock a libvirt VM off the LAN and tailnet](../runbooks/lock-a-libvirt-vm-off-the-lan.md).

## Lessons
- Test isolation from inside the guest. Do not trust the diagram in your head.
- This VM has passwordless sudo, so anything running as its user gets root with no prompt. That makes the network boundary the control that matters most.

## Related Notes
- [vm-lockdown](../services/vm-lockdown.md)
- [Lock a libvirt VM off the LAN and tailnet](../runbooks/lock-a-libvirt-vm-off-the-lan.md)
- [NAT is not isolation](../concepts/nat-is-not-isolation.md)

## Tags
#incident
