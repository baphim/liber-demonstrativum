---
status: Running
host: "caddy LXC"
reviewed: 2026-10-20
tags: [service]
---

# Caddy

## Overview
| Field | Value |
|-------|-------|
| **Purpose** | Internal reverse proxy. Routes subdomains of `baphim.xyz` to backend hosts and terminates TLS. |
| **Host** | caddy LXC, see [Infrastructure](../infrastructure.md) |
| **Status** | Running |

## Deployment
Custom build rather than stock Caddy, to bundle a DNS-01 plugin for the domain registrar. It was first built with `xcaddy`, and is now kept current with `caddy add-package` and `caddy upgrade`. Current routes:

| Subdomain | TLS method |
|---|---|
| `vaultwarden.baphim.xyz` | Real cert, Let's Encrypt via DNS-01. Tailnet only since October 2026. |
| `nextcloud.baphim.xyz` | Real cert, Let's Encrypt via DNS-01. Tailnet only since October 2026. |
| `grafana.baphim.xyz` | Self-signed internal CA, intentionally LAN/tailnet-only |
| `git.baphim.xyz` | Real cert, Let's Encrypt via DNS-01. Intentionally public, hosts a coursework repo shared with a professor. |

The split isn't arbitrary: public-facing services get real publicly-trusted certs, and services that only need to be reachable inside the lab can use Caddy's self-signed internal CA. A real cert does not mean a service is public. Vaultwarden keeps its real cert after leaving the public internet, because DNS-01 never needed it to be reachable. See [DNS-01 vs HTTP-01](../concepts/dns-01-vs-http-01.md) and [reverse proxy TLS termination](../concepts/reverse-proxy-tls-termination.md).

## Configuration
Uses a DNS-01 challenge plugin for the domain registrar's API. The credentials live in a root-only environment file (mode 600) loaded by the service through `EnvironmentFile=`, not as `Environment=` lines in the unit. Unit-file values are printed by `systemctl status` when the service fails to start, which is how the first set leaked, see [the incident](../incidents/caddy-apt-upgrade-removed-dns-module.md).

The package is held with `apt-mark hold caddy`, because an `apt upgrade` replaces the custom binary with the stock one, which lacks the DNS plugin. Updates are manual: `caddy upgrade`, or unhold, upgrade and re-run `caddy add-package`.

## Dependencies
- Depends on the registrar's API being reachable for certificate issuance and renewal.
- Depends on the custom binary staying in place. A stock binary cannot parse the Caddyfile.
- Is the front door for [Vaultwarden](vaultwarden.md), [Nextcloud](nextcloud.md), Grafana, and Gitea.

## Known Issues
- Not every self-hosted service goes through Caddy, some are reached directly by LAN IP instead, and one dashboard resolves via a local DNS record rather than a Caddyfile entry. An intentional mix, not every internal tool needs a public-style proxy entry.
- Held out of apt on purpose, so it gets no automatic security updates. Checking for new releases is a manual job, and this LXC is the front door for the lab.
- An `apt upgrade` once replaced the custom build with the stock binary and took every proxied service down. Documented in [the incident](../incidents/caddy-apt-upgrade-removed-dns-module.md).
- The LXC's disk was originally sized for a plain reverse proxy, not for also hosting a Go toolchain to build `xcaddy`. Ran out of space once as a result, documented and fixed (see incidents).
- Caddy logs a harmless, expected error on every reload as long as one domain still uses the self-signed internal CA, it's Caddy trying to install its own local CA root into the container and failing. Worth remembering so it isn't mistaken for a real problem next time the logs are checked for something unrelated.

## Related Notes
- [Infrastructure](../infrastructure.md)
- [Vaultwarden](vaultwarden.md)
- [Nextcloud](nextcloud.md)
- [Caddy apt upgrade removed the DNS provider module](../incidents/caddy-apt-upgrade-removed-dns-module.md)
- [Caddy LXC out of disk space](../incidents/caddy-disk-full.md)
- [Convert a Proxmox LXC from DHCP to static IP](../runbooks/dhcp-to-static-ip.md)

## Tags
#service
