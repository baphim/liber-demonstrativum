---
tags:
  - runbook
  - "#prometheus"
---

# Install node_exporter and wire it into Prometheus

Based on adding metrics coverage for several hosts that had been missed when this lab's monitoring was first set up. See [Prometheus](../services/prometheus.md) for the resulting scrape config.

## Step 1: Check for a distro package first
Debian/Ubuntu ships node_exporter as a package, prefer this over a manual install when available:
```
apt-get update
apt-get install -y prometheus-node-exporter prometheus-node-exporter-collectors
```
The second package provides the textfile collector scripts, useful if this host will ever write custom metrics. See [the textfile collector pattern](../concepts/textfile-collector.md).

**Gotcha**: the installed service is named `prometheus-node-exporter`, not `node_exporter`, and the running process shows as `prometheus-node` in `ps` output due to Linux's 15-character `comm` field truncation. Searching for `node_exporter` by name will come back empty even though it's installed and running correctly. Use the real service name.

On other distros (Amazon Linux, RHEL-family, anything without the package), check first:
```
dnf search node_exporter
```
If nothing comes back, go to Step 2.

## Step 2: Manual binary install (when no package exists)
```
sudo bash -c '
VERSION=$(curl -s https://api.github.com/repos/prometheus/node_exporter/releases/latest | grep -oP "\"tag_name\": \"v\K[^\"]+")
cd /tmp
curl -sLO "https://github.com/prometheus/node_exporter/releases/download/v${VERSION}/node_exporter-${VERSION}.linux-amd64.tar.gz"
tar -xzf "node_exporter-${VERSION}.linux-amd64.tar.gz"
cp "node_exporter-${VERSION}.linux-amd64/node_exporter" /usr/local/bin/
useradd --no-create-home --shell /usr/sbin/nologin node_exporter 2>/dev/null
chown node_exporter:node_exporter /usr/local/bin/node_exporter
mkdir -p /var/lib/prometheus/node-exporter
chown node_exporter:node_exporter /var/lib/prometheus/node-exporter
'
```
Looks up the current release automatically rather than hardcoding a version that goes stale.

Create the systemd unit:
```
sudo bash -c 'cat > /etc/systemd/system/node_exporter.service << EOF
[Unit]
Description=Prometheus Node Exporter
After=network.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter --collector.textfile.directory=/var/lib/prometheus/node-exporter

[Install]
WantedBy=multi-user.target
EOF'
sudo systemctl daemon-reload
sudo systemctl enable --now node_exporter
sudo systemctl status node_exporter --no-pager
```
The `--collector.textfile.directory` flag matches the path used by the distro package elsewhere, keep it consistent so any textfile-based metrics work the same way regardless of which install method a given host used.

## Step 3: Open the firewall, if the host has one
If the target host runs firewalld or similar with a default-deny policy, add a rule scoped to the actual source that needs it, not a blanket allow:
```
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="<CIDR>" port port="9100" protocol="tcp" accept'
sudo firewall-cmd --reload
sudo firewall-cmd --list-rich-rules
```
Scope the source CIDR to wherever Prometheus itself actually scrapes from, matching the pattern of existing rules on that host rather than introducing a new style.

## Step 4: Add the Prometheus scrape job
On the Prometheus host, edit `/etc/prometheus/prometheus.yml` and add a job:
```yaml
  - job_name: "<name>"
    static_configs:
      - targets: ["<ip>:9100"]
        labels:
          app: "<name>"
```
Reload:
```
systemctl reload prometheus
```

## Step 5: Confirm the target is actually healthy
Don't just trust the config, check Prometheus's own view of it:
```
curl -s "http://<prometheus-ip>:9090/api/v1/targets" | python3 -c "
import json, sys
data = json.load(sys.stdin)
for t in data['data']['activeTargets']:
    if t['labels'].get('job') == '<name>':
        print(t['health'], t.get('lastError', ''))
"
```
`up` with no error means it's working. `down` with `context deadline exceeded` almost always means a firewall is silently dropping the scrape, revisit Step 3. `unknown` usually just means Prometheus hasn't attempted its first scrape yet.

## Notes / gotchas
- Always check for a distro package before doing a manual install.
- The `prometheus-node-exporter` naming trap in Step 1 is worth remembering broadly, not just for node_exporter, any time a service search comes up empty on a Debian-family host, check the actual package name rather than assuming it's not installed.
- A `down` target with a timeout error, not an immediate connection error, is the signature of a firewall dropping (not rejecting) the traffic.

## Related Notes
- [Prometheus](../services/prometheus.md)
