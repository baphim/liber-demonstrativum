---
tags:
  - concept
  - "#proxmox"
  - "#linux"
---

# Unprivileged LXC confinement (cgroups vs AppArmor)

An unprivileged container on a hypervisor is one that does not hold any actual privilege on its host. This functions because the root user of the container (UID 0) is assigned a very high UID on the host itself (i.e. UID 100000). This configuration has effects that can be unrecognized during setup. For instance, Tailscale requires a special device file `/dev/net/tun`, known as a TUN device, to build its VPN interface.

There are two separate controls that prevent this device from being created on an unprivileged container: the cgroup device allowlist and AppArmor. The cgroup allowlist is the **narrower** of the two, it does exactly one thing, decide which specific device files (by major:minor number) a container's processes are permitted to touch. AppArmor is the **broader** control, a general-purpose mandatory access control system governing file access, capabilities, mounts, and more, device access is just one small slice of what it restricts.

Adding the container to the cgroup allowlist on the host permits device-level access specifically, but with AppArmor still confining the container more broadly, the device creation was still blocked even after the narrower cgroup permission was granted. Both had to be addressed, the cgroup rule to allow the device itself, and loosening AppArmor's confinement (`lxc.apparmor.profile: unconfined`) for the broader set of operations actually needed to create and use it.

## Related Notes
- [Proxmox](../services/proxmox.md)
- [Enable Tailscale in an unprivileged Proxmox LXC](../runbooks/enable-tailscale-unprivileged-lxc.md)
- [Prometheus tailscale join](../incidents/prometheus-tailscale-join.md)

## Tags
#concept
