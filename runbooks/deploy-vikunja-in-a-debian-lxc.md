---
tags:
  - runbook
  - "#proxmox"
  - "#tailscale"
---

# Deploy Vikunja in a Debian LXC (tailnet only)

A native `.deb` install in an unprivileged Debian 12 LXC, reachable only over the tailnet with HTTPS, monitored by Wazuh and covered by backups. Values in angle brackets are placeholders. See [Vikunja](../services/vikunja.md) for the finished service.

## Step 1: Pre-checks on the Proxmox host
Confirm the container ID and IP are free and the template exists.
```
pct list
ip route | grep default
pveam list local
```

## Step 2: Create the container
Put the root filesystem on the snapshot-capable ZFS pool. `nesting=1` is needed because Debian 12's systemd in an unprivileged container otherwise warns at first start and may break service sandboxing later.
```
pct create <ctid> local:vztmpl/<debian-12-template>.tar.zst \
  --hostname vikunja \
  --cores 1 --memory 1024 --swap 512 \
  --rootfs data:8 \
  --unprivileged 1 \
  --features nesting=1 \
  --net0 name=eth0,bridge=vmbr0,ip=<ip>/24,gw=<gateway> \
  --onboot 1
```

## Step 3: Add TUN passthrough, then start
Do this before any snapshot exists, or the lines land in a snapshot section. The reasoning is in [Enable Tailscale in an unprivileged Proxmox LXC](enable-tailscale-unprivileged-lxc.md).
```
cat >> /etc/pve/lxc/<ctid>.conf <<'EOF'
lxc.cgroup2.devices.allow: c 10:200 rwm
lxc.mount.entry: /dev/net/tun dev/net/tun none bind,create=file
EOF
pct start <ctid>
pct enter <ctid>
```

## Step 4: Update and install prerequisites
```
apt update && apt full-upgrade -y
apt install -y curl ca-certificates
```

## Step 5: Find and install the newest release
Ignore the `unstable` directory and use the newest numbered one.
```
curl -s https://dl.vikunja.io/vikunja/ | grep -o 'href="[^"]*"' | sort -V | tail -15
curl -s https://dl.vikunja.io/vikunja/<version>/ | grep -o 'href="[^"]*"' | grep -i deb
cd /tmp
curl -LO https://dl.vikunja.io/vikunja/<version>/vikunja-<version>-x86_64.deb
apt install -y ./vikunja-<version>-x86_64.deb
```
The package installs enabled but not running. Config is `/etc/vikunja/config.yml`, data is under `/opt/vikunja/`.

## Step 6: Write the config
Use `service.secret`, since `JWTSecret` is deprecated in v2. Start with the LAN URL for testing. The `tr: write error: Broken pipe` message is harmless, so confirm the secret length instead.
```
cp /etc/vikunja/config.yml /etc/vikunja/config.yml.orig
SECRET=$(tr -dc 'A-Za-z0-9' </dev/urandom 2>/dev/null | head -c 48)
echo ${#SECRET}
cat > /etc/vikunja/config.yml <<EOF
service:
  secret: "$SECRET"
  publicurl: "http://<ip>:3456/"
  enableregistration: true
database:
  type: "sqlite"
  path: "/opt/vikunja/vikunja.db"
EOF
chmod 600 /etc/vikunja/config.yml
unset SECRET
```
The length should print `48`. Do not paste the config file anywhere, because it contains the secret.

## Step 7: Start, check, and create the account
```
systemctl start vikunja
journalctl -u vikunja --no-pager -n 30
curl -s http://localhost:3456/api/v1/info | head -c 300
```
Expect warnings about no license key and no mailer. Create the account in a browser at `http://<ip>:3456`.

## Step 8: Close registration and take a checkpoint
```
sed -i 's/enableregistration: true/enableregistration: false/' /etc/vikunja/config.yml
systemctl restart vikunja
```
From the host:
```
pct snapshot <ctid> fresh-install --description "Vikunja installed, registration closed"
```

## Step 9: Install Tailscale and serve over HTTPS
Enable HTTPS certificates for the tailnet first, on the DNS page of the admin console.
```
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up --hostname=vikunja
tailscale serve --bg 3456
tailscale serve status
```

## Step 10: Point publicurl at the tailnet URL
Keep the trailing slash. It must match what the web UI and the Android app use, because it drives the CORS allowlist.
```
sed -i 's|publicurl: ".*"|publicurl: "https://vikunja.<tailnet>.ts.net/"|' /etc/vikunja/config.yml
systemctl restart vikunja
journalctl -u vikunja --no-pager -n 5 | grep -i cors
```

## Step 11: Install the Wazuh agent, pinned
The agent must not be newer than the manager. Get the manager's version first with `/var/ossec/bin/wazuh-control info` on the manager.
```
apt install -y gnupg
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | gpg --no-default-keyring --keyring gnupg-ring:/usr/share/keyrings/wazuh.gpg --import && chmod 644 /usr/share/keyrings/wazuh.gpg
echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main" > /etc/apt/sources.list.d/wazuh.list
apt update
apt-cache madison wazuh-agent | head -5
WAZUH_MANAGER="<manager-address>" WAZUH_AGENT_NAME="vikunja" apt install -y wazuh-agent=<manager-version>-1
systemctl daemon-reload
systemctl enable --now wazuh-agent
apt-mark hold wazuh-agent
```
A `netstat not available` message from the syscheck daemon is harmless. Install `net-tools` if the listening-port check is wanted.

## Step 12: Add it to the backup job
In the Proxmox UI under Datacenter, then Backup.

## Verification
- `systemctl is-active vikunja` returns `active`.
- `curl -s http://localhost:3456/api/v1/info` returns the expected version.
- `tailscale serve status` shows the HTTPS URL proxying to `http://127.0.0.1:3456`.
- The web login and the Android app both work with the tailnet URL.
- The CORS log line lists the tailnet origin, and the create-account link is gone from the login page.
- `apt-mark showhold` lists `wazuh-agent`, and the agent shows Active on the manager.
- The container appears in the next vzdump run.

## Related Notes
- [Vikunja](../services/vikunja.md)
- [Enable Tailscale in an unprivileged Proxmox LXC](enable-tailscale-unprivileged-lxc.md)
- [Build FreshRSS in a Proxmox LXC](build-freshrss-in-lxc.md)

## Tags
#runbook
