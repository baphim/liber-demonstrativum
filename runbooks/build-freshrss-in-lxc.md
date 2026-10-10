---
tags:
  - runbook
  - "#proxmox"
---

# Build FreshRSS in a Proxmox LXC

Native install (Apache, PHP, SQLite) in an unprivileged Debian 13 LXC, with a local DNS name, Tailscale inside the container, a pinned Wazuh agent, nightly backups and a weekly OPML export. Values in angle brackets are placeholders. See [FreshRSS](../services/freshrss.md) for the finished service.

Prerequisites: a free container ID, a free static IP, a `zfspool` storage for the container disk (not a directory storage, since snapshots matter), a backup-capable storage, a reachable Pi-hole, and the Wazuh manager version.

## Step 1: Create the container
Put the disk on the ZFS pool storage, not the backup directory. A directory storage fails with `does not support container directories`.
```
pct list
ls /etc/pve/lxc/ | grep <ctid>
pveam update
pveam available --section system | grep debian-13
pveam download local debian-13-standard_<version>_amd64.tar.zst

pct create <ctid> local:vztmpl/debian-13-standard_<version>_amd64.tar.zst \
  --hostname rss \
  --unprivileged 1 \
  --cores 1 \
  --memory 1024 \
  --swap 512 \
  --rootfs <zfs-storage>:8 \
  --net0 name=eth0,bridge=vmbr0,ip=<rss-ip>/24,gw=<gateway> \
  --nameserver <pihole-ip> \
  --searchdomain <domain> \
  --features nesting=1 \
  --onboot 1 \
  --password
```

## Step 2: Add TUN passthrough before starting or snapshotting
Tailscale in an unprivileged LXC needs `/dev/net/tun`. Add the lines now, before any `pct snapshot`. Anything appended after a snapshot header lands in the snapshot section and not the live config. This is the same trap described in [Prometheus tailscale join](../incidents/prometheus-tailscale-join.md), and I hit it again on this build.
```
ls -l /dev/net/tun        # on the host; run modprobe tun if missing
cat >> /etc/pve/lxc/<ctid>.conf <<'EOF'
lxc.cgroup2.devices.allow: c 10:200 rwm
lxc.mount.entry: /dev/net/tun dev/net/tun none bind,create=file
EOF
```
If a snapshot already exists, rebuild the file with the two lines above the first `[` header. `/etc/pve` does not like in-place `sed -i`, so write the new file in `/root` and copy it back.

## Step 3: Start and update
```
pct start <ctid>
pct exec <ctid> -- ls -l /dev/net/tun
pct exec <ctid> -- ping -c2 deb.debian.org
pct exec <ctid> -- bash -c 'apt update && apt -y full-upgrade'
pct enter <ctid>
```
The remaining steps run inside the container as root unless noted.

## Step 4: Install packages
```
apt install -y apache2 php php-sqlite3 php-curl php-xml php-mbstring php-zip php-intl php-gd php-json curl unzip cron ca-certificates
php -v
```
Optional, to silence perl locale warnings: install `locales`, uncomment `en_US.UTF-8` in `/etc/locale.gen`, run `locale-gen` and `update-locale LANG=en_US.UTF-8`.

## Step 5: Download FreshRSS
Look up the current release instead of reusing an old tag.
```
curl -s https://api.github.com/repos/FreshRSS/FreshRSS/releases/latest | grep '"tag_name"'
cd /tmp
curl -L -o freshrss.tar.gz https://github.com/FreshRSS/FreshRSS/archive/refs/tags/<tag>.tar.gz
file freshrss.tar.gz
mkdir -p /var/www/FreshRSS
tar -xzf freshrss.tar.gz --strip-components=1 -C /var/www/FreshRSS
chown -R www-data:www-data /var/www/FreshRSS
chmod -R g+w /var/www/FreshRSS/data /var/www/FreshRSS/extensions
```
`file` must say gzip compressed data. `ls /var/www/FreshRSS` should show `app`, `cli`, `data`, `extensions`, `p`.

## Step 6: Apache vhost
```
cat > /etc/apache2/sites-available/freshrss.conf <<'EOF'
<VirtualHost *:80>
    ServerName <fqdn>
    DocumentRoot /var/www/FreshRSS/p/

    <Directory /var/www/FreshRSS/p>
        AllowOverride AuthConfig FileInfo Indexes Limit
        Require all granted
    </Directory>

    <Directory /var/www/FreshRSS/p/api>
        AllowOverride AuthConfig FileInfo Indexes Limit
    </Directory>

    AllowEncodedSlashes On
    ErrorLog ${APACHE_LOG_DIR}/freshrss_error.log
    CustomLog ${APACHE_LOG_DIR}/freshrss_access.log combined
</VirtualHost>
EOF

a2enmod rewrite headers expires env dir mime
a2dissite 000-default
a2ensite freshrss
apache2ctl configtest
systemctl restart apache2
systemctl is-active apache2
```
Use `restart`, not `reload`, since `a2enmod` enables new modules. A warning about the server's fully qualified domain name is harmless.

## Step 7: CLI install and user
The CLI install avoids needing browser access to the container.
```
cd /var/www/FreshRSS
runuser -u www-data -- ./cli/do-install.php \
  --default-user <username> \
  --auth-type form \
  --environment production \
  --base-url http://<fqdn> \
  --language en \
  --title FreshRSS \
  --api-enabled \
  --db-type sqlite

read -rs -p "Login password: " PW; echo
read -rs -p "API password: " APW; echo
runuser -u www-data -- ./cli/create-user.php --user <username> --password "$PW" --api-password "$APW"
unset PW APW

./cli/access-permissions.sh
chown -R www-data:www-data /var/www/FreshRSS
```
The prompts keep both passwords out of shell history. The API password is separate from the login password, and the reader app uses the API one. Both need at least 7 characters. Older docs spell the flag `--api_password`, which is deprecated.

## Step 8: Feed refresh cron
```
cat > /etc/cron.d/freshrss <<'EOF'
*/20 * * * * www-data php /var/www/FreshRSS/app/actualize_script.php > /tmp/FreshRSS.log 2>&1
EOF
chmod 644 /etc/cron.d/freshrss
chmod 644 /etc/crontab /etc/cron.d/e2scrub_all
systemctl enable --now cron
```
The second `chmod` matters. The template ships those files group-writable, which makes cron log `INSECURE MODE` and can stop it running them. Check with `journalctl -u cron --no-pager -n 5`.

## Step 9: DNS record
In the Pi-hole web UI, add an A record under Local DNS Records pointing at the container IP, then check it.
```
dig +noall +answer +comments <fqdn> @<pihole-ip>
```
If the answer is a stale upstream result (`EDE: 3 (Stale Answer)`, or a parking page from the public zone's wildcard), restart FTL on the Pi-hole host and query again.
```
sudo systemctl restart pihole-FTL
```

## Step 10: Tailscale
```
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up --hostname=rss
tailscale ip -4
```
Approve the node from the login URL. In the admin console, turn on MagicDNS and disable key expiry for the node, otherwise sync stops after about 180 days. Off the LAN, use the tailnet name or IP, since the local DNS name only resolves on the LAN.

## Step 11: Wazuh agent
The agent must not be newer than the manager.
```
# on the manager
/var/ossec/bin/wazuh-control info | grep WAZUH_VERSION

# in the container: check the manager ports first
timeout 3 bash -c '</dev/tcp/<wazuh-ip>/1515' && echo "1515 open"
timeout 3 bash -c '</dev/tcp/<wazuh-ip>/1514' && echo "1514 open"

apt install -y gnupg apt-transport-https
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | gpg --no-default-keyring --keyring gnupg-ring:/usr/share/keyrings/wazuh.gpg --import && chmod 644 /usr/share/keyrings/wazuh.gpg
echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main" > /etc/apt/sources.list.d/wazuh.list
apt update
apt-cache madison wazuh-agent | head
WAZUH_MANAGER="<wazuh-ip>" WAZUH_AGENT_NAME="rss" apt install -y wazuh-agent=<manager-version>-1
apt-mark hold wazuh-agent
systemctl daemon-reload
systemctl enable --now wazuh-agent
```
The repo was one patch version ahead of the manager, which is why the version is pinned. Run `apt-mark unhold wazuh-agent` before upgrading the manager.

## Step 12: OPML export script
```
install -d -o www-data -g www-data -m 750 /var/backups/freshrss-opml

cat > /usr/local/bin/freshrss-opml-export.sh <<'EOF'
#!/bin/bash
set -euo pipefail
cd /
OUT=/var/backups/freshrss-opml
FILE="$OUT/freshrss-<username>-$(date +%F).opml"
/var/www/FreshRSS/cli/export-opml-for-user.php --user <username> > "$FILE.tmp"
grep -q '<opml' "$FILE.tmp"
mv "$FILE.tmp" "$FILE"
find "$OUT" -name 'freshrss-<username>-*.opml' -mtime +90 -delete
EOF
chmod 755 /usr/local/bin/freshrss-opml-export.sh

cat > /etc/cron.d/freshrss-opml <<'EOF'
0 3 * * 0 www-data /usr/local/bin/freshrss-opml-export.sh
EOF
chmod 644 /etc/cron.d/freshrss-opml

runuser -u www-data -- /usr/local/bin/freshrss-opml-export.sh
echo "exit code: $?"
```
The `cd /` line matters. `runuser` keeps the caller's working directory, and `find` fails if `www-data` cannot enter it. Without that line the export works but the script exits 1.

## Step 13: Backup job (on the Proxmox host)
Run it once by hand first, then create the job. Check the existing job list so the new job does not overlap an existing one.
```
vzdump <ctid> --storage <backup-storage> --mode snapshot --compress zstd
pvesh get /cluster/backup --output-format json-pretty
pvesh create /cluster/backup \
  --id rss-nightly \
  --schedule "02:30" \
  --storage <backup-storage> \
  --vmid <ctid> \
  --mode snapshot \
  --compress zstd \
  --prune-backups keep-daily=7,keep-weekly=4,keep-monthly=3 \
  --notes-template '{{guestname}}' \
  --enabled 1
```
The job's own retention overrides the storage default. If the backup storage shares a pool with the container, add an off-pool copy. That part is still open for this build.

## Step 14: Import the feeds
Check every feed URL against the live site before building the OPML. Some sites block bots, so a few cannot be confirmed by fetch and need a first-fetch check in FreshRSS. Import under Subscription management, Import / export, then open each unconfirmed feed and make sure it has articles.

## Step 15: Reader app
In the Android reader (FeedMe worked), add a FreshRSS or Google Reader API account. Server URL is `http://<fqdn>/api/greader.php`, with the username and the API password. The phone must be on the tailnet. If an app refuses plain http, put HTTPS in front with `tailscale serve`.

## Verification
- `pct exec <ctid> -- ls -l /dev/net/tun` shows a character device `10, 200`.
- `curl -sI http://127.0.0.1/i/ | head -n 3` inside the container returns 200.
- `/tmp/FreshRSS.log` shows a clean run after a :00, :20 or :40 mark.
- `journalctl -u cron --no-pager -n 5` shows no `INSECURE MODE` lines.
- `dig +noall +answer <fqdn> @<pihole-ip>` returns a single A record with the container IP.
- `apt-mark showhold` lists `wazuh-agent`, and the manager lists the agent as active.
- `pvesh get /cluster/backup` lists the new job, and `/var/backups/freshrss-opml/` holds a dated `.opml` after the script runs.

## Related Notes
- [FreshRSS](../services/freshrss.md)
- [Enable Tailscale in an unprivileged LXC](enable-tailscale-unprivileged-lxc.md)
- [Install node_exporter](install-node-exporter.md)
