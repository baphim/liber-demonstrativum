---
tags:
  - runbook
  - "#proxmox"
  - "#tailscale"
  - "#ntfy"
---

# Deploy ntfy in a Debian LXC (tailnet only)

A private ntfy server in an unprivileged Debian LXC, reachable only over the tailnet with HTTPS, and a phone subscribed to it. Values in angle brackets are placeholders. See [ntfy](../services/ntfy.md) for the finished service.

## Why self-host
ntfy is a server plus a phone app. I had only the app at first, so nothing could push. The public ntfy.sh server needs no setup, but there the topic name is the only secret. I used it for one test to prove the phone worked, then self-hosted.

## Step 1: Test the phone against public ntfy.sh
Subscribe the app to a long random topic and publish from any host.
```
curl -d "test" https://ntfy.sh/<long-random-topic>
```
If it arrives, the phone and the app are fine before you touch your own server.

## Step 2: Create the container
Create an unprivileged Debian LXC, 1 core and 512 MB is enough. Tailscale in an unprivileged container needs TUN passthrough, so add these two lines to the container's config on the Proxmox host (`/etc/pve/lxc/<ctid>.conf`):
```
lxc.cgroup2.devices.allow: c 10:200 rwm
lxc.mount.entry: /dev/net/tun dev/net/tun none bind,create=file
```
Inside the container, `ls -l /dev/net/tun` should show a character device (the line starts with `c`). Nesting is not needed. If ntfy ever fails with `status=226/NAMESPACE`, turning nesting on is the fix.

## Step 3: Install ntfy from the official apt repository
```
dpkg --print-architecture
apt update && apt install -y curl apt-transport-https
mkdir -p /etc/apt/keyrings
curl -L -o /etc/apt/keyrings/ntfy.gpg https://archive.ntfy.sh/apt/keyring.gpg
echo "deb [arch=amd64 signed-by=/etc/apt/keyrings/ntfy.gpg] https://archive.ntfy.sh/apt stable main" > /etc/apt/sources.list.d/ntfy.list
apt update
apt install -y ntfy
```
Check the key with `gpg --show-keys /etc/apt/keyrings/ntfy.gpg` and compare the fingerprint to the one in the ntfy docs. The package installs the service disabled, which is what you want before configuring it.

## Step 4: Configure it as private and localhost only
```
cp /etc/ntfy/server.yml /etc/ntfy/server.yml.orig
cat > /etc/ntfy/server.yml <<'CONF'
base-url: "https://ntfy.<tailnet>.ts.net"
listen-http: "127.0.0.1:2586"
behind-proxy: true
cache-file: "/var/cache/ntfy/cache.db"
auth-file: "/var/lib/ntfy/user.db"
auth-default-access: "deny-all"
CONF
systemctl enable --now ntfy
journalctl -u ntfy -n 20 --no-pager
```
- It listens on localhost only because Tailscale puts HTTPS in front of it.
- `behind-proxy: true` is required behind a proxy. Without it, rate limiting treats every visitor as one.
- `deny-all` makes the server private. The default is open to everyone.

Create the admin user as the `ntfy` account:
```
runuser -u ntfy -- ntfy user add --role=admin <admin-user>
```
> **Gotcha:** Running `ntfy user add` as root can leave the user database owned by root, and then the service may not be able to write to it.

Test both sides:
```
curl -i -d "no auth" http://127.0.0.1:2586/test
curl -i -u <admin-user> -d "with auth" http://127.0.0.1:2586/test
```
The first should return 403 and the second 200.

## Step 5: Tailscale and HTTPS
Install Tailscale in the container and run `tailscale up`. Then put HTTPS in front of ntfy:
```
tailscale serve --bg 2586
```
`--bg` keeps it running and it resumes after a reboot. HTTPS certificates have to be enabled on the tailnet first. Check from another tailnet device:
```
curl -i https://ntfy.<tailnet>.ts.net/v1/health
```
It should return 200 with `{"healthy":true,"role":"active"}`.

> **Gotcha:** My first health check failed with `curl: (77) error adding trust anchors from file`, pointing at a build directory. A conda `base` environment was active in my shell, and its curl has a stale CA path. Calling `/usr/bin/curl` directly worked. I did not fix the conda curl.

## Step 6: Phone
Connect Tailscale on the phone, then subscribe in the ntfy app. Use your tailnet URL as the server, your admin login, and a topic name. Publish a test from another device. Lock the phone for a couple of minutes and send a second one, to check it arrives while the phone sleeps.

> **Gotcha:** With a self-hosted server on Android there is no Firebase push. The app keeps its own connection open, so Tailscale has to stay on and the ntfy app's battery setting has to be unrestricted. `upstream-base-url` in the ntfy config is for iOS only.

## Step 7: One write-only bot user per sender
```
runuser -u ntfy -- ntfy user add <bot-user>
runuser -u ntfy -- ntfy access <bot-user> <topic> write-only
runuser -u ntfy -- ntfy token add --label="<what uses it>" <bot-user>
runuser -u ntfy -- ntfy access
```
An ntfy token gives full access to its user account, so the point is a user that can only write to one topic. If the token leaks, it cannot read messages or touch any other topic. `ntfy access` should show your admin user as admin, the bot with write-only on its one topic, and no access for anonymous.

Test from the sender with a silent prompt, so the token never lands in shell history:
```
read -rs NTFY_TOKEN
curl -H "Authorization: Bearer $NTFY_TOKEN" -d "test" https://ntfy.<tailnet>.ts.net/<topic>
unset NTFY_TOKEN
```
This also proves the sender can reach the server over the tailnet.

## Step 8: Keep the node from expiring
In the Tailscale admin console, disable key expiry for the ntfy node. Otherwise the server drops off the tailnet when the key expires and notifications stop with no warning. I put this on my list. I did not record that I did it.

## Related Notes
- [ntfy](../services/ntfy.md)
- [Enable Tailscale in an unprivileged Proxmox LXC](enable-tailscale-unprivileged-lxc.md)
- [Send a Vikunja digest to ntfy](send-a-vikunja-digest-to-ntfy.md)
- [Send Wazuh alerts to ntfy](send-wazuh-alerts-to-ntfy.md)

## Tags
#runbook
