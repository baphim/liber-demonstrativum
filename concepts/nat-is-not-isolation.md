---
tags:
  - concept
  - "#networking"
  - "#libvirt"
---

# NAT is not isolation

NAT rewrites addresses so a private network can share one outside address. It is often described as keeping machines hidden from outside, and that is true in one direction only. Replies to connections the VM started are allowed back in, and unsolicited inbound connections from the outside have no mapping to land on. Nothing in that stops the VM from starting connections to whatever the NAT gateway itself can reach.

## How it works
A libvirt `default` network puts VMs on a private bridge and masquerades their outbound traffic through the host. To a device on my LAN, the VM looks like the workstation. That has two consequences:

- The LAN cannot start connections to the VM. This is the part people remember.
- The VM can start connections to anything the host can route to, including the LAN, anything reachable through the host's VPN or tailnet interface, and any service on the host that listens on the bridge.

Isolation is a policy: an explicit rule that says which destinations the VM may not reach. NAT is addressing. They happen to overlap in one direction.

## In my setup
I built a Kali Purple VM on the workstation's default network believing it was off my LAN. It was not. It could reach a LAN device, a tailnet device and the workstation's SSH. I fixed it with a firewall rule that rejects forwarded traffic from the bridge to the private and tailnet ranges, and by removing SSH from the firewalld zone that libvirt uses. See [Kali Purple VM exposed to the LAN and tailnet](../incidents/kali-purple-vm-lan-tailnet-exposure.md) and [vm-lockdown](../services/vm-lockdown.md).

A reject rule in its own nftables table works well here, because a reject or drop verdict is final. Other chains, including the ones libvirt and firewalld add, cannot override it by accepting later.

The Proxmox Kali VM sits on a separate virtual bridge with no route to the rest of the LAN. That is isolation by topology, and it is the pattern I use for deliberately vulnerable targets.

## Related Notes
- [vm-lockdown](../services/vm-lockdown.md)
- [Lock a libvirt VM off the LAN and tailnet](../runbooks/lock-a-libvirt-vm-off-the-lan.md)

## Tags
#concept
