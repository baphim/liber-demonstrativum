# Index

Every page in this vault, grouped by purpose. The folders stay flat so links to individual pages keep working.

Start with [Infrastructure](infrastructure.md) for what is running and on what.

## Services

### Platform

- [Proxmox](services/proxmox.md)
- [Pi-hole](services/pi-hole.md)
- [Caddy](services/caddy.md)
- [Tailscale](services/tailscale.md)

### Applications

- [Vaultwarden](services/vaultwarden.md)
- [Nextcloud](services/nextcloud.md)
- [Gitea](services/gitea.md)
- [FreshRSS](services/freshrss.md)
- [Vikunja](services/vikunja.md)
- [ntfy](services/ntfy.md)

### Monitoring and security

- [Wazuh](services/wazuh.md)
- [Prometheus](services/prometheus.md)
- [Grafana](services/grafana.md)
- [Suricata](services/suricata.md)
- [Zeek](services/zeek.md)
- [vm-lockdown](services/vm-lockdown.md)
- [Kali](services/kali.md)
- [Metasploitable](services/metasploitable.md)

### Local models

- [Ollama](services/ollama.md)
- [Local models](services/local-models.md)

## Incidents

- [Caddy apt upgrade removed the DNS provider module](incidents/caddy-apt-upgrade-removed-dns-module.md)
- [Caddy LXC out of disk space](incidents/caddy-disk-full.md)
- [Kali Purple VM exposed to the LAN and tailnet](incidents/kali-purple-vm-lan-tailnet-exposure.md)
- [Prometheus tailscale join, TUN device and lxc.conf snapshot bug](incidents/prometheus-tailscale-join.md)
- [Proxmox host DNS root cause (tailscale accept-dns)](incidents/proxmox-dns-root-cause.md)
- [Wazuh alert review: a local model flagged four alert groups](incidents/wazuh-alert-review.md)
- [Wazuh dashboard error, disk full, vulnerability detector bloat](incidents/wazuh-disk-full.md)
- [Wazuh vulnerability audit after the 4.14.9 upgrade](incidents/wazuh-vulnerability-audit.md)

## Runbooks

### Build and deploy

- [Build FreshRSS in a Proxmox LXC](runbooks/build-freshrss-in-lxc.md)
- [Deploy Vikunja in a Debian LXC (tailnet only)](runbooks/deploy-vikunja-in-a-debian-lxc.md)
- [Deploy ntfy in a Debian LXC (tailnet only)](runbooks/deploy-ntfy-in-a-debian-lxc.md)
- [Deploy a Wazuh manager from OVA](runbooks/deploy-wazuh-ova.md)
- [Set up Ollama](runbooks/set-up-ollama.md)

### Network and access

- [Convert a Proxmox LXC from DHCP to static IP](runbooks/dhcp-to-static-ip.md)
- [Enable Tailscale in an unprivileged Proxmox LXC](runbooks/enable-tailscale-unprivileged-lxc.md)
- [Take a service off the public internet](runbooks/take-a-service-off-the-public-internet.md)
- [Set up key-only SSH to a libvirt VM](runbooks/set-up-key-only-ssh-to-a-libvirt-vm.md)
- [Lock a libvirt VM off the LAN and tailnet](runbooks/lock-a-libvirt-vm-off-the-lan.md)

### Monitor and triage

- [Run Wazuh alert triage with a local model](runbooks/run-wazuh-alert-triage.md)
- [Verify a rootcheck trojan alert](runbooks/verify-a-rootcheck-trojan-alert.md)
- [Check a VPS for SSH exposure](runbooks/check-vps-ssh-exposure.md)
- [Upgrade Wazuh central components and agents](runbooks/upgrade-wazuh-central-and-agents.md)
- [Fix Wazuh vulnerability-detector disk bloat](runbooks/fix-wazuh-disk-bloat.md)
- [Send Wazuh alerts to ntfy](runbooks/send-wazuh-alerts-to-ntfy.md)
- [Send a Vikunja digest to ntfy](runbooks/send-a-vikunja-digest-to-ntfy.md)
- [Install node_exporter and wire it into Prometheus](runbooks/install-node-exporter.md)
- [Add a local Suricata rule](runbooks/add-a-local-suricata-rule.md)
- [Capture traffic with Zeek in a podman container](runbooks/capture-traffic-with-zeek-in-podman.md)
- [Take a baseline of a fresh VM](runbooks/take-a-vm-baseline.md)

### Update, back up and repair

- [Update Nextcloud from the command line](runbooks/update-nextcloud-from-the-cli.md)
- [Update Vaultwarden built from source](runbooks/update-vaultwarden-from-source.md)
- [Vaultwarden backup restore procedure](runbooks/vaultwarden-backup-restore.md)
- [Fix an apt repo failing on a SHA1 key signature](runbooks/fix-apt-repo-sha1-key-error.md)

## Concepts

- [Detection in code, explanation in the model](concepts/detection-in-code-explanation-in-the-model.md)
- [DNS-01 vs HTTP-01 (ACME challenges)](concepts/dns-01-vs-http-01.md)
- [LSM-trees (RocksDB)](concepts/lsm-trees-rocksdb.md)
- [Models, agents and weights](concepts/models-agents-and-weights.md)
- [NAT is not isolation](concepts/nat-is-not-isolation.md)
- [Java RMI](concepts/java-rmi.md)
- [Unsafe deserialization](concepts/unsafe-deserialization.md)
- [Reverse proxy TLS termination](concepts/reverse-proxy-tls-termination.md)
- [The textfile collector pattern](concepts/textfile-collector.md)
- [Unprivileged LXC confinement (cgroups vs AppArmor)](concepts/unprivileged-lxc-confinement.md)
- [WAL mode in SQLite (and why backups use `.backup`)](concepts/wal-mode-sqlite.md)
