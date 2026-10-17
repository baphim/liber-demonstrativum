---
date: 2026-10-16
status: Resolved
tags:
  - incident
  - "#caddy"
  - "#secrets"
---

# Caddy apt upgrade removed the DNS provider module

## What Happened
I ran `apt upgrade` on the Caddy LXC. Caddy did not come back up, and I could no longer edit Vaultwarden entries from the Bitwarden Android app, which reported a failed connection to port 443 on the Caddy container's tailnet address. Everything proxied through Caddy was unreachable while it was down.

A second problem showed up while I was troubleshooting. The DNS provider's API key and secret printed in plain text in the `systemctl status caddy` output, and I pasted that output into an AI chat session. I rotated the keys before fixing Caddy.

## Investigation
1. The app error showed the name resolving to a tailnet address. So DNS and the tailnet were working, and the failure was the connection to port 443 on the Caddy side. That pointed at Caddy not listening, not at the phone.

2. `systemctl status caddy` showed `Active: failed (Result: exit-code)` with `status=1/FAILURE`. The last log line before the exit was `using config from file`, so Caddy died while loading the Caddyfile. The output was cut off at the environment dump from the unit, so it did not show the actual error. That dump is also where the keys leaked, which is why I stopped to rotate them before going further.

3. I checked whether the DNS provider module was still in the binary, since the Caddyfile uses it for the DNS-01 challenge:
```
caddy list-modules | grep -i <provider>
```
It returned nothing. The module was gone.

4. I validated the Caddyfile to get the real error:
```
caddy validate --config /etc/caddy/Caddyfile
```
```
ERROR   adapting config using caddyfile: parsing caddyfile tokens for 'tls': getting module named 'dns.providers.<provider>': module not registered
```
This ran from my shell without the environment variables loaded, which did not matter. The failure was the missing module, which happens before any key is read.

## Root Cause
`apt upgrade` replaced my custom Caddy binary with the stock one from the Caddy package. The stock build does not include the DNS provider module, and the Caddyfile uses it in a `tls` block. With the module missing, the Caddyfile could not be parsed, so Caddy exited with status 1 and never opened port 443. The Caddyfile itself had not changed.

apt owns the binary path, so every package upgrade writes the stock file over any custom build. A module added with `caddy add-package` does not survive an `apt upgrade`.

The key exposure had a separate cause. The keys were stored as `Environment=` lines in a systemd drop-in, and the failed start printed the service environment into the journal, which `systemctl status` displayed.

## Fix
### 1. Rotate the keys
The old pair was exposed in terminal output and in a chat, and a DNS API key can modify DNS, so I replaced it first.

1. In the provider's dashboard I created a new pair and deleted the old one. I also confirmed API access was enabled on the domain, since this provider requires that per domain no matter which key is used.
2. I prompted for the new keys silently, so they stay out of shell history and off the screen:
```
read -rsp 'API key: ' K; echo
read -rsp 'Secret key: ' S; echo
```
3. I wrote them to a root-only env file. `install -m 600` creates the file with the right mode before the keys go in, so there is never a moment where it is readable by anyone else:
```
install -m 600 -o root -g root /dev/null /etc/caddy/dns.env
printf 'DNS_API_KEY=%s\nDNS_SECRET_KEY=%s\n' "$K" "$S" > /etc/caddy/dns.env
unset K S
```
4. I verified the file without printing the secrets. It showed `-rw------- 1 root root` and only the two variable names:
```
ls -l /etc/caddy/dns.env
cut -d= -f1 /etc/caddy/dns.env
```
5. I removed the two `Environment=` lines from the drop-in and pointed it at the env file instead. This keeps the keys out of the unit file, where `systemctl status` and `systemctl show` can display them:
```
[Service]
EnvironmentFile=/etc/caddy/dns.env
```
6. I reloaded systemd and checked that it picked up the file and that the old values were gone from the unit. The first command listed the env file and the second printed `0`:
```
systemctl daemon-reload
systemctl show caddy -p EnvironmentFiles
systemctl show caddy -p Environment | grep -c <variable-prefix>
```
7. I cleared the old keys from the journal. They were already revoked, so this was for good measure:
```
journalctl --rotate && journalctl --vacuum-time=1s
```

### 2. Restore the module
`caddy add-package` downloads a rebuilt binary with the module from Caddy's build server and swaps it in over the stock one:
```
caddy add-package github.com/caddy-dns/<provider>
caddy list-modules | grep -i <provider>
```
I validated again with the variables loaded into my shell, since only systemd loads the env file on its own:
```
set -a; . /etc/caddy/dns.env; set +a
caddy validate --config /etc/caddy/Caddyfile
```
The validate passed, so I started the service. Caddy came up and Vaultwarden was reachable again.

### 3. Stop apt from overwriting it again
I held the package so future `apt upgrade` runs skip it:
```
apt-mark hold caddy
```
The tradeoff is that Caddy no longer gets security updates through apt. To update, either run `caddy upgrade`, which fetches the latest version with the added modules, or run `apt-mark unhold caddy`, upgrade, and re-run `caddy add-package`. Checking for Caddy updates is now a manual job, and since this LXC is the front door for the lab, it needs to actually happen.

## Lessons
- A custom binary installed over a package-managed path is not safe from the package manager. Hold the package.
- Secrets in `Environment=` lines are printed by `systemctl status` on a failed start. Use `EnvironmentFile=` with mode 600 instead.
- Treat terminal output as public the moment it might be pasted somewhere. Rotate first, then debug.

## Related Notes
- [Caddy](../services/caddy.md)
- [Vaultwarden](../services/vaultwarden.md)
- [Caddy LXC out of disk space](caddy-disk-full.md)

## Tags
#incident
