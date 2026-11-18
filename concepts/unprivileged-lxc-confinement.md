---
tags:
  - concept
  - "#proxmox"
  - "#linux"
---

# Unprivileged LXC confinement (cgroups vs AppArmor)

An unprivileged container on a hypervisor holds no real privilege on its host. That works because the container's root user (UID 0) maps to a very high UID on the host itself (for example, UID 100000). It has side effects that aren't obvious until you hit one: Tailscale needs a special device file, `/dev/net/tun`, to build its VPN interface, and creating that device is exactly the kind of thing an unprivileged container is restricted from doing.

Two separate controls block it: the cgroup device allowlist and AppArmor. The cgroup allowlist is the narrower of the two. It does one thing, decide which specific device files (by major:minor number) a container's processes can touch. AppArmor is the broader control, a general-purpose mandatory access control system covering file access, capabilities, mounts, and more. Device access is just one small slice of what it restricts.

Adding the container to the cgroup allowlist grants device-level access specifically, but AppArmor was still confining the container more broadly, so device creation stayed blocked even after that. Both had to be fixed: the cgroup rule to allow the device itself, and loosening AppArmor's confinement (`lxc.apparmor.profile: unconfined`) for the broader set of operations actually needed to create and use it.

## Related Notes
- [Proxmox](../services/proxmox.md)
- [Enable Tailscale in an unprivileged Proxmox LXC](../runbooks/enable-tailscale-unprivileged-lxc.md)
- [Prometheus tailscale join](../incidents/prometheus-tailscale-join.md)

## Tags
#concept
