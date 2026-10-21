---
status: Running
host: "prometheus LXC"
tags: [service]
---

# Prometheus

## Overview
| Field | Value |
|-------|-------|
| **Purpose** | Metrics collection, feeding [Grafana](grafana.md)'s dashboards |
| **Host** | prometheus LXC, see [Infrastructure](../infrastructure.md) |
| **Status** | Running |

## Configuration
Node exporter scrapes every host in the lab, one job per machine, standard node_exporter port.

**Coverage gap closed**: Prometheus, Grafana, and the Wazuh server originally had no scrape target of their own, simply never installed, not a deliberate exclusion. Fixed by installing the distro package where available (Debian hosts), and a manual binary install where it wasn't (Amazon Linux). One practical gotcha from that fix: the Debian package names its service `prometheus-node-exporter`, not `node_exporter`, and the running process shows as a truncated name in `ps` due to Linux's 15-character limit on process names, worth knowing before assuming a service search coming up empty means it isn't installed. See [Install node_exporter and wire it into Prometheus](../runbooks/install-node-exporter.md).

**Pushgateway**: present, confirmed unused. Its own metrics endpoint shows only self-monitoring counters, nothing from an actual application. Likely set up anticipating a need that ended up being solved a different way.

**Backup status metrics, two different designs**:
- Vaultwarden's backup script writes its own success/timestamp metrics directly as part of the backup run. See [Vaultwarden](vaultwarden.md).
- Proxmox's weekly backup job uses a separate checker script on its own timer that queries Proxmox's own task log for the most recent backup job and reports based on its actual status, rather than hooking into the backup job directly. One real gap here: the check runs on a fixed schedule rather than after the job actually finishes, so an unusually long-running backup could be read as stale and still report success.

Both patterns use node_exporter's textfile collector, a lightweight way to get custom metrics into Prometheus from a one-shot script without writing a full exporter. See [the textfile collector pattern](../concepts/textfile-collector.md).

**Blackbox exporter probes**: internal reachability checks by IP for LAN-only services, and public reachability checks by domain for the services that are genuinely internet-facing, confirmed each has a real, valid certificate.

## Dependencies
- [Grafana](grafana.md) uses this as its data source, confirmed directly against Grafana's own database.

## Known Issues
- Joined the tailnet later than the rest of the fleet, previously LAN-only. The join itself required working through an AppArmor/TUN-device confinement issue and surfaced a real Proxmox config-file bug along the way, see [the incident writeup](../incidents/prometheus-tailscale-join.md).
- Not yet proxied through Caddy, a deliberate sequencing choice (the tailnet join was the priority), not a rejected idea.

## Related Notes
- [Infrastructure](../infrastructure.md)
- [Grafana](grafana.md)
- [Install node_exporter and wire it into Prometheus](../runbooks/install-node-exporter.md)
- [Enable Tailscale in an unprivileged Proxmox LXC](../runbooks/enable-tailscale-unprivileged-lxc.md)

## Tags
#service
