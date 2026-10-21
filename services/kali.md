---
status: Running
host: "Kali VM"
reviewed: 2026-10-20
tags: [service]
---

# Kali

## Overview
| Field | Value |
|-------|-------|
| **Purpose** | Attacker box for pentesting practice against the isolated Metasploitable target |
| **Host** | Dedicated VM, see [Infrastructure](../infrastructure.md) |
| **Status** | Active, powered off between sessions |

This is a different machine from the Kali Purple VM on the workstation, which is a defensive-tools sandbox, not an attacker box. See [vm-lockdown](vm-lockdown.md).

## Access
No LAN route, no tailnet, internet access only. It sits on a separate virtual bridge, routed so it can only reach [Metasploitable](metasploitable.md), never the rest of the LAN. I removed Tailscale from this VM to keep it isolated during pentesting.

## Deployment
Standard Kali Linux VM on Proxmox, with Metasploit and its database, nmap and the usual Kali toolset.

## Dependencies
- [Proxmox](proxmox.md)

## Known Issues
- A Wazuh agent is installed but shows disconnected while the VM is powered off, which is most of the time.
- There is an unused disk attached to the VM that I don't remember the purpose of.

## Related Notes
- [Metasploitable](metasploitable.md)
- [Java RMI](../concepts/java-rmi.md)
- [Unsafe deserialization](../concepts/unsafe-deserialization.md)

## Tags
#service
