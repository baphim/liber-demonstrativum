---
status: Retired
tags: [service]
---

# Valheim

## Overview
| Field | Value |
|-------|-------|
| **Purpose** | Solo play, heavily modded Valheim server, built to move the load off a personal workstation |
| **Status** | Retired, decommissioned the same day it was built |

## Deployment
- LXC built via the Proxmox web UI, with nesting and `keyctl` enabled to allow Docker to run unprivileged.
- Server run via a `docker-compose.yml` using a community Valheim server image, which handled the game server install itself.

## Configuration
- Mods came from a local modding tool profile. Its export format only carried a fraction of the installed mods correctly, so the actual plugin folder was copied over directly instead of relying on the export.
- Hit the same DNS inheritance issue as every other freshly-created LXC, needed manual correction on creation.
- Ran into an out-of-memory condition during modded play and needed the LXC's memory resized before it would run reliably.

## Outcome
Built specifically to test moving Valheim off the workstation and onto the homelab. In actual use, resource usage under a heavily modded solo profile was heavy enough that it wasn't worth keeping running long-term. Shut down and the LXC destroyed shortly after, a deliberate decision rather than something that broke.

## Tags
#service
