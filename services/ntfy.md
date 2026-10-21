---
status: Running
host: "ntfy LXC"
reviewed: 2026-10-20
tags: [service]
---

# ntfy

## Overview
| Field | Value |
|-------|-------|
| **Purpose** | Self-hosted push notifications to my phone: a daily task digest from [Vikunja](vikunja.md) and high-severity alerts from [Wazuh](wazuh.md) |
| **Host** | Dedicated unprivileged LXC, see [Infrastructure](../infrastructure.md) |
| **Status** | Running |

I tried the public ntfy.sh server first, only to prove the phone and the app worked. On the public server the topic name is the only secret, so I decided to host it myself.

## Access
Tailnet only. The server listens on localhost, and `tailscale serve` puts HTTPS in front of it, so there is no [Caddy](caddy.md) entry and no public route. The phone has to stay connected to Tailscale to receive notifications.

## Deployment
Installed from the official ntfy apt repository into an unprivileged Debian LXC with TUN passthrough for Tailscale. I checked the repository key's fingerprint against the ntfy docs before trusting it. Steps: [Deploy ntfy in a Debian LXC](../runbooks/deploy-ntfy-in-a-debian-lxc.md).

## Configuration
- `listen-http` is on localhost only, `behind-proxy` is true, and `auth-default-access` is `deny-all`, so nothing is open to anonymous users.
- One admin user for me, and one write-only bot user per sender. Each bot can publish to its own topic and nothing else, so a leaked bot token cannot read messages or touch other topics.
- `ntfy user` commands run as the `ntfy` user, not root, so the user database does not end up owned by root.
- Senders: [a Vikunja digest](../runbooks/send-a-vikunja-digest-to-ntfy.md) from a systemd timer, and [Wazuh alerts](../runbooks/send-wazuh-alerts-to-ntfy.md) at level 10 and up from a custom integration.
- Wazuh agent installed and pinned to the manager's version.

## Dependencies
- Tailscale, for access and for HTTPS certificates
- [Proxmox](proxmox.md)

## Known Issues
- I did not check whether the first scheduled digest arrived.
- There is no Firebase on a self-hosted Android setup, so the phone keeps its own connection open. The ntfy app's battery setting has to be unrestricted.
- If the node's Tailscale key expires, the server drops off the tailnet and notifications stop without any warning. Turning key expiry off was on my list, and I did not record that I did it.
- The Wazuh integration has no deduplication, so a burst of alerts means many pings.

## Related Notes
- [Deploy ntfy in a Debian LXC](../runbooks/deploy-ntfy-in-a-debian-lxc.md)
- [Send a Vikunja digest to ntfy](../runbooks/send-a-vikunja-digest-to-ntfy.md)
- [Send Wazuh alerts to ntfy](../runbooks/send-wazuh-alerts-to-ntfy.md)

## Tags
#service
