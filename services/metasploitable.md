---
status: Running
host: "Metasploitable VM"
reviewed: 2026-10-20
tags: [service]
---

# Metasploitable

## Overview
| Field | Value |
|-------|-------|
| **Purpose** | Deliberately vulnerable target for pentesting practice against Kali |
| **Host** | Dedicated VM, see [Infrastructure](../infrastructure.md) |
| **Status** | Active, powered off between sessions |

Metasploitable 2 is a Linux image built to have known, intentionally unpatched vulnerabilities. I never update or patch it, on purpose.

## Access
No LAN route, no tailnet, no internet. It shares an isolated segment with [Kali](kali.md) only, and nothing outside that segment can reach it.

## Deployment
Standard Metasploitable 2 image on Proxmox, running its default set of deliberately vulnerable services.

## Dependencies
- [Proxmox](proxmox.md)

## Known Issues
- No Wazuh agent, by design, since the box stays frozen and unpatched.
- I tried monitoring it through a syslog relay from Kali. That relay never worked, and I haven't replaced it with anything.

## Related Notes
- [Kali](kali.md)
- [Java RMI](../concepts/java-rmi.md)
- [Unsafe deserialization](../concepts/unsafe-deserialization.md)

## Tags
#service
