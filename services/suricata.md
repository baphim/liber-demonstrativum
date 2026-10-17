---
status: Running
host: "[Infrastructure](../infrastructure.md)"
tags: [service]
---

# Suricata

## Overview
| Field | Value |
|-------|-------|
| **Purpose** | Network IDS watching the Kali Purple VM's interface, for practice reading alerts and writing rules |
| **Host** | Kali Purple VM on the workstation, see [Infrastructure](../infrastructure.md) |
| **Status** | Running |

This is a practice sandbox, separate from the lab's monitoring. It does not feed [Wazuh](wazuh.md). A production IDS fed by a switch mirror port is planned separately.

## Access
No listening port. Everything is read from log files on the VM.

## Deployment
Installed from the Kali repo (8.0.7), enabled with systemd. Rules come from `suricata-update`, which pulled the Emerging Threats Open set (53,211 rules loaded, 0 failed). Rules take about 30 seconds to load after a restart. Adding my own rules: [Add a local Suricata rule](../runbooks/add-a-local-suricata-rule.md).

## Configuration
- Interface: set in the `af-packet` section of `suricata.yaml`, pointing at the VM's only NIC.
- Merged rules are built by `suricata-update`. Local rules live in their own file and are listed under `local:` in `update.yaml`. Without that entry, a plain `suricata-update` drops them.
- Test rule: `sid:1000001` alerts on an HTTP user agent containing `curl/`.
- Logs: `fast.log` (one line per alert), `eve.json` (full JSON events), `suricata.log` and `stats.log`.
- `suricata -T` tests the config. `suricata-update` does not restart the service, so restart after rule changes.

## Dependencies
- Internet access for rule downloads, which [vm-lockdown](vm-lockdown.md) still allows.

## Known Issues
- It only sees the VM's own traffic on the NAT network, so live alerts are limited to what that VM does. Most practice material will be sample pcaps, which I have not run yet.
- The usual `testmynids.org` test URL no longer exists (NXDOMAIN). My local rule replaces it.
- `suricata-update --dump-sample-configs` writes six sample files into the current directory instead of printing them.
- It only reads plain HTTP content. HTTPS shows up as TLS metadata.

## Related Notes
- [Zeek](zeek.md)
- [vm-lockdown](vm-lockdown.md)
- [Add a local Suricata rule](../runbooks/add-a-local-suricata-rule.md)
- [Take a baseline of a fresh VM](../runbooks/take-a-vm-baseline.md)

## Tags
#service
