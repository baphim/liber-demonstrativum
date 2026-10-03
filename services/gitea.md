---
status: Running
host: "[Infrastructure](../infrastructure.md)"
tags: [service]
---

# Gitea

## Overview
| Field | Value |
|-------|-------|
| **Purpose** | Self-hosted git server, source for several private repos, mirrors out to GitHub as a backup |
| **Host** | [Infrastructure](../infrastructure.md) |
| **Status** | Running |

## Access
`git.baphim.xyz`, proxied by [Caddy](caddy.md). Real Let's Encrypt cert via DNS-01.

## Configuration
Mirrors private repos out to GitHub as a backup, confirmed private on the GitHub side as well, so internal infrastructure detail documented in those repos isn't exposed through the mirror.

One repo is intentionally public: coursework shared with a university instructor for review. Per-repo visibility is set deliberately, that one repo is public, everything else on this instance is private.

## Dependencies
- Depends on [Pi-hole](pi-hole.md) for DNS resolution, has broken once when that pointed elsewhere, see Known Issues.
- Depends on GitHub being reachable for the outbound mirror push.
- Reachable through [Caddy](caddy.md).

## Known Issues
- The public coursework repo briefly served a self-signed certificate rather than a publicly-trusted one, since it was originally set up assuming LAN/tailnet-only access. Caught and fixed, switched to a real cert matching the other public-facing services, confirmed working from outside the network.
- Same DNS fragility as other hosts: a Proxmox-level DNS misconfiguration once broke the GitHub mirror until the root cause was found and fixed at the source. See [the incident writeup](../incidents/proxmox-dns-root-cause.md).

## Related Notes
- [Infrastructure](../infrastructure.md)
- [Pi-hole](pi-hole.md)
- [Proxmox host DNS root cause](../incidents/proxmox-dns-root-cause.md)

## Tags
#service
