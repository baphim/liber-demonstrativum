---
status: Running
host: "a dedicated LXC"
reviewed: 2026-11-14
tags: [service]
---

# spyderbox

## Overview
| Field | Value |
|-------|-------|
| **Purpose** | Remote Spyder desktop for Python coursework, used from a tablet with a keyboard away from my desk |
| **Host** | A dedicated unprivileged LXC, see [Infrastructure](../infrastructure.md) |
| **Status** | Running |

## Access
RDP only, there's no web page. Reachable over the tailnet address, not the LAN address, even though the RDP server listens on both, see Configuration. Login is the Linux account's own password. There's no sudo on that account.

## Deployment
A lightweight desktop environment and Spyder from the distro's own repo, served by an RDP server, inside an unprivileged LXC with TUN passthrough for Tailscale. I tried a browser-based remote desktop first and dropped it, mostly because it didn't handle the tablet's screen well. Steps: [Set up spyderbox](../runbooks/set-up-spyderbox.md).

## Configuration
- The desktop session starts from a per-user session file that names the desktop environment to launch.
- The RDP login is the account's normal Linux password, not a separate credential.
- The display's DPI needed raising by hand to be readable on the tablet's high-density screen; the RDP client's own scaling options didn't do anything for this server.
- A project repo is cloned in, with one shared virtual environment at the repo root rather than one per subproject, since the two machines involved run different Python versions and pinning would break between them.
- Git credentials for pushing are stored in plaintext locally, so they get revoked whenever this container is retired.
- Firewall: a ruleset of its own inside the container, default-drop on input, explicitly accepting loopback, established connections, everything on the tailnet interface, ICMP, and Tailscale's own port. The RDP server still listens on every address, but the port is closed on the LAN address and open only on the tailnet address. See [Proxmox firewall and tailnet traffic](../concepts/proxmox-firewall-and-tailnet-traffic.md) for why a hypervisor-level rule can't do this job here.

## Dependencies
- [Proxmox](proxmox.md)
- [Tailscale](tailscale.md)
- [Gitea](gitea.md), for the project repo

## Known Issues
- At the DPI needed for a readable desktop, the application menu becomes too large to use comfortably. Worked around with a keyboard shortcut and a panel launcher instead.
- A crashed session can leave a stale lock file that makes the IDE report it's already running with no window visible. The fix is clearing that lock, not killing a process match on the IDE's name, since on a host whose own hostname contains that name, a process search can match an unrelated system service instead.
- No monitoring agent installed on this guest yet.
- The two machines involved run different Python point releases. Left as is until it causes a real problem.

## Related Notes
- [Infrastructure](../infrastructure.md)
- [Set up spyderbox](../runbooks/set-up-spyderbox.md)
- [Proxmox firewall and tailnet traffic](../concepts/proxmox-firewall-and-tailnet-traffic.md)
- [Decommission an LXC](../runbooks/decommission-an-lxc.md)

## Tags
#service
