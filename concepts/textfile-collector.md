---
tags:
  - concept
  - "#prometheus"
---

# The textfile collector pattern

A one-shot script can report a status to Prometheus without running its own web server. It writes a `.prom` file on completion or failure, which node_exporter's textfile collector picks up.

The textfile collector isn't a separate continuously-running thing, it's a feature built into node_exporter itself, the same node_exporter that's already running on every host to expose system metrics. There's no extra process involved, and nothing gets pushed. The script writes the file, node_exporter merges its contents into the normal `/metrics` output, and Prometheus scrapes it on its usual interval, same as any other metric.

## Why this matters here
This is the mechanism behind both backup-status metrics on the Grafana dashboard, Vaultwarden's backup script and the Proxmox backup checker script both write directly to a `.prom` file rather than talking to Prometheus or the pushgateway at all.

## Related Notes
- [Vaultwarden](../services/vaultwarden.md)
- [Prometheus](../services/prometheus.md)
- [Grafana](../services/grafana.md)

## Tags
#concept
