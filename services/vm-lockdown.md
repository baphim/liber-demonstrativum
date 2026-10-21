---
status: Running
host: "workstation"
tags: [service]
---

# vm-lockdown

## Overview
| Field | Value |
|-------|-------|
| **Purpose** | Keeps VMs on the workstation's libvirt `default` network off the LAN, the tailnet and the host's own services, while leaving internet access |
| **Host** | Workstation, see [Infrastructure](../infrastructure.md) |
| **Status** | Running |

## Deployment
An nftables table, `inet vm_lockdown`, loaded by a oneshot systemd unit that runs after `libvirtd`. The unit's stop action deletes the table. It is enabled, and I confirmed the lockdown was still in place after rebooting the workstation. Steps: [Lock a libvirt VM off the LAN and tailnet](../runbooks/lock-a-libvirt-vm-off-the-lan.md).

## Configuration
- A forward-hook rule (priority -10) rejects traffic arriving from the libvirt bridge and headed to the private ranges, the tailnet range and link-local. The internet stays open.
- In firewalld, `ssh` and `tftp` are removed from the `libvirt` zone, so VMs cannot reach the host's SSH. DHCP and DNS stay allowed.
- Stopping the unit lifts the lockdown, and starting it (or a reboot) puts it back.

## Dependencies
- libvirt, for the bridge.
- firewalld, for the zone change.
- nftables, the `nft` binary only.

## Known Issues
- It applies to every VM on the `default` network, not just the Kali Purple VM. That is on purpose, so future VMs there are covered.
- It matches the bridge by name, so a VM on any other libvirt network is not covered.
- While it is stopped, a VM can reach everything the workstation can, including the tailnet. Snapshot first and keep those sessions short.
- The VM can still resolve internal names through the host's DNS, but it cannot connect to anything they point to.
- Enabling the distro's `nftables.service` would remove the table, because its default config starts with `flush ruleset`.

## Related Notes
- [Kali Purple VM exposed to the LAN and tailnet](../incidents/kali-purple-vm-lan-tailnet-exposure.md)
- [Lock a libvirt VM off the LAN and tailnet](../runbooks/lock-a-libvirt-vm-off-the-lan.md)
- [NAT is not isolation](../concepts/nat-is-not-isolation.md)
- [Suricata](suricata.md)
- [Zeek](zeek.md)

## Tags
#service
