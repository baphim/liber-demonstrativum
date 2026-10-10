---
status: Running
host: "[Infrastructure](../infrastructure.md)"
tags: [service]
---

# Wazuh

## Overview
| Field | Value |
|-------|-------|
| **Purpose** | SIEM / security monitoring for every host in the lab |
| **Host** | [Infrastructure](../infrastructure.md) |
| **Status** | Running |

## Access
| Field | Value |
|-------|-------|
| **URL** | `wazuh.baphim.xyz`, resolved via a local DNS record, not published externally |
| **Ports** | 443 (dashboard), 1514 (agent event data), 1515 (agent enrollment), 55000 (API) |

## Deployment
- Deployed from Wazuh's prebuilt OVA rather than a manual install.
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

## Dependencies
- Depends on [Pi-hole](pi-hole.md) for local DNS resolution.
- Depends on the tailnet for agents outside the LAN segment to reach the manager.
- Agents deployed to every LXC by LAN IP, plus the workstation and Kali by tailnet IP.

## Known Issues
- Running on 2 vCPU against the documented recommendation of 4. No problems observed, but genuinely untested under real alert load.
- Email alerting isn't configured yet. Wazuh doesn't handle SMTP auth itself, so this needs a separate OS-level mail relay, parked mid-setup.
- An agent must not be newer than the manager. The FreshRSS container's agent is pinned to the manager's version with `apt-mark hold`, so run `apt-mark unhold wazuh-agent` before upgrading the manager.
- Metasploitable intentionally has no agent, it's a frozen, deliberately vulnerable target. Monitored instead via a syslog relay through Kali.
- A known upstream Wazuh bug causes the vulnerability-detector module's RocksDB feed to accumulate without cleanup, eventually filling the disk entirely, which took the dashboard down. Root cause, fix, and two real operational gotchas (the right way to stop wazuh-manager, and a `rm -rf` glob-expansion trap) are documented in [the incident writeup](../incidents/wazuh-disk-full.md) and [the fix runbook](../runbooks/fix-wazuh-disk-bloat.md). Likely to recur, worth checking periodically rather than waiting for a repeat.
- The VM's disk has very little margin once the vulnerability feed regrows. A deliberate resize is worth considering alongside just watching for the bloat.

## Related Notes
- [Infrastructure](../infrastructure.md)
- [Deploy a Wazuh manager from OVA](../runbooks/deploy-wazuh-ova.md)
- [Fix Wazuh vulnerability-detector disk bloat](../runbooks/fix-wazuh-disk-bloat.md)
- [Wazuh dashboard error, disk full, vulnerability detector bloat](../incidents/wazuh-disk-full.md)

## Tags
#service
