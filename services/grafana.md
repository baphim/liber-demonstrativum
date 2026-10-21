---
status: Running
host: "grafana LXC"
reviewed: 2026-10-20
tags: [service]
---

# Grafana

## Overview
| Field | Value |
|-------|-------|
| **Purpose** | Dashboards, visualizing Prometheus data |
| **Host** | grafana LXC, see [Infrastructure](../infrastructure.md) |
| **Status** | Running |

## Access
Proxied by [Caddy](caddy.md), served via Caddy's self-signed internal CA rather than a publicly-trusted cert, consistent with being LAN/tailnet-only. It was reachable through the VPS until October 2026. That route and its public DNS record are gone.

## Dashboards
- **Node Exporter Full**: a prebuilt community dashboard, visualizing the node_exporter scrapes from every host.
- **Backups** (custom): tracks two separate backup jobs, each with the same three-panel pattern: a status indicator, a last-run timestamp, and a pass/fail history timeline.
  - **Proxmox's weekly backup**: fed by a checker script that queries Proxmox's own task log rather than hooking into the backup job directly.
  - **Vaultwarden's daily backup**: fed directly by the backup script itself as part of its run.
  Both use node_exporter's textfile collector pattern. Full detail in [Prometheus](prometheus.md) and [Vaultwarden](vaultwarden.md).

## Configuration
One data source configured (Prometheus), added by hand through the UI rather than provisioned via config file. Since it was added manually, it won't survive a rebuild of this host from scratch, worth provisioning it properly at some point so that's not a manual step to redo.

## Dependencies
- [Prometheus](prometheus.md), confirmed as its data source.

## Known Issues
- Uses Caddy's self-signed internal CA. Confirmed no browser warning when loading the dashboard by domain name on a tested device, so the CA is trusted there. Direct IP access bypasses Caddy and its cert entirely, so it isn't a valid way to test this.
- The Prometheus data source was added by hand, not provisioned. A host rebuild loses it, has to be redone manually.

## Related Notes
- [Infrastructure](../infrastructure.md)
- [Prometheus](prometheus.md)
- [Caddy](caddy.md)
- [Convert a Proxmox LXC from DHCP to static IP](../runbooks/dhcp-to-static-ip.md)
- [Install node_exporter and wire it into Prometheus](../runbooks/install-node-exporter.md)

## Tags
#service
