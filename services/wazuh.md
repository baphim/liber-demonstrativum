---
status: Running
host: "wazuh-server VM"
reviewed: 2026-10-20
tags: [service]
---

# Wazuh

## Overview
| Field | Value |
|-------|-------|
| **Purpose** | SIEM / security monitoring for every host in the lab |
| **Host** | wazuh-server VM, see [Infrastructure](../infrastructure.md) |
| **Status** | Running |

## Access
| Field | Value |
|-------|-------|
| **URL** | `wazuh.baphim.xyz`, resolved via a local DNS record, not published externally |
| **Ports** | 443 (dashboard), 1514 (agent event data), 1515 (agent enrollment), 55000 (API) |

## Deployment
- Deployed from Wazuh's prebuilt OVA rather than a manual install. Currently 4.14.9, all in one (indexer, manager, Filebeat, dashboard).
- 2 vCPU, 8GB RAM. Wazuh's documented recommendation is 4 vCPU, running on 2 since that was what the host could spare, untested under real alert load.
- Disk on a ZFS pool for snapshot support.
- Base OS is Amazon Linux 2023, later upgraded to a newer release.
- Joined to the tailnet, no public internet access.

See [Deploy a Wazuh manager from OVA](../runbooks/deploy-wazuh-ova.md) for the full build process.

## Configuration
- Static LAN IP set via `systemd-networkd`, since the OVA's networking is AL2023-native rather than Debian-style.
- SSH key auth required switching from Ed25519 to RSA. The OVA ships in FIPS mode, which disallows Ed25519 keys.
- firewalld restricts inbound access to LAN and tailnet source ranges only, default policy is deny. Rich rules scoped per source/port pair, standard manager ports for both ranges, plus the metrics port for LAN only.
- A local DNS record resolves the dashboard hostname for LAN clients.
- Alerts at level 10 and up are pushed to my phone through a custom integration and a self-hosted [ntfy](ntfy.md) server, using a write-only token. Steps: [Send Wazuh alerts to ntfy](../runbooks/send-wazuh-alerts-to-ntfy.md).

## Dependencies
- Depends on [Pi-hole](pi-hole.md) for local DNS resolution.
- Depends on the tailnet for agents outside the LAN segment to reach the manager.
- Agents connect to the manager over the tailnet. The one exception is the Proxmox host, which enrolls over the LAN.
- The Kali Purple VM on the workstation is deliberately not enrolled. It is a sandbox with its own [Suricata](suricata.md) and [Zeek](zeek.md), and it is blocked from the tailnet, so it could not reach the manager anyway.

## Known Issues
- Running on 2 vCPU against the documented recommendation of 4. No problems observed, but genuinely untested under real alert load.
- Email alerting isn't configured yet. Wazuh doesn't handle SMTP auth itself, so this needs a separate OS-level mail relay, parked mid-setup.
- An agent must not be newer than the manager. Every agent is held at the manager's version (`apt-mark hold`, or `IgnorePkg` on the workstation). Upgrade the manager first, then unhold, upgrade and re-hold each agent.
- Metasploitable intentionally has no agent, it's a frozen, deliberately vulnerable target. I tried to monitor it through a syslog relay on Kali, but the relay never worked.
- A known upstream Wazuh bug causes the vulnerability-detector module's RocksDB feed to accumulate without cleanup, eventually filling the disk entirely, which took the dashboard down. Root cause, fix, and two real operational gotchas (the right way to stop wazuh-manager, and a `rm -rf` glob-expansion trap) are documented in [the incident writeup](../incidents/wazuh-disk-full.md) and [the fix runbook](../runbooks/fix-wazuh-disk-bloat.md). Likely to recur, worth checking periodically rather than waiting for a repeat.
- The disk was too small for the 4.14.9 vulnerability scanner. I grew it from 25G to 50G in October 2026. Free space still needs watching.

- **Some default rules are noisy for this setup.** Rule 5712 (SSH brute force) fires at level 10 against a server with key-only auth, where guessing cannot succeed. Rule 510 (rootcheck trojan) gave a false positive on a legitimate mail client, multiplied by symlinked `/bin` and `/sbin` directories. Rule 533 (listening ports changed) fires on `tailscaled`'s temporary UDP ports. Planned fixes are a local rule to lower 5712 for that host, and filtering the ephemeral ports in the agent's netstat command instead of a rule, since a rule matching `tailscaled` would hide real port changes. See [the alert review](../incidents/wazuh-alert-review.md).
- A local model can explain alerts, but its output has to be checked against the raw alert. See [Run Wazuh alert triage with a local model](../runbooks/run-wazuh-alert-triage.md).

## Related Notes
- [Infrastructure](../infrastructure.md)
- [Deploy a Wazuh manager from OVA](../runbooks/deploy-wazuh-ova.md)
- [Fix Wazuh vulnerability-detector disk bloat](../runbooks/fix-wazuh-disk-bloat.md)
- [Wazuh dashboard error, disk full, vulnerability detector bloat](../incidents/wazuh-disk-full.md)
- [Wazuh alert review](../incidents/wazuh-alert-review.md)
- [Run Wazuh alert triage with a local model](../runbooks/run-wazuh-alert-triage.md)
- [Verify a rootcheck trojan alert](../runbooks/verify-a-rootcheck-trojan-alert.md)
- [Check a VPS for SSH exposure](../runbooks/check-vps-ssh-exposure.md)

## Tags
#service
