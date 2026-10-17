---
status: Stopped
host: "[Infrastructure](../infrastructure.md)"
tags: [service]
---

# Zeek

## Overview
| Field | Value |
|-------|-------|
| **Purpose** | Network traffic analyzer that logs connections, DNS, HTTP and more, for practice reading protocol logs |
| **Host** | Kali Purple VM on the workstation, see [Infrastructure](../infrastructure.md) |
| **Status** | Stopped. Run by hand for a capture, not kept as a service |

Like [Suricata](suricata.md), this is a practice sandbox that does not feed [Wazuh](wazuh.md).

## Deployment
Kali's apt package (5.1.1) does not install on the current release. It depends on `libc6 (< 2.38)` while the VM has 2.43, and on a library that is no longer available. So Zeek runs from its official container image under podman, which was Zeek 9.0.0 when pulled. Podman has no daemon, all its unit files are disabled, and nothing new listens on the network, so the VM's baseline holds. The image is stored twice, once for my user and once for root, because `sudo podman` has its own storage. Steps: [Capture traffic with Zeek in a podman container](../runbooks/capture-traffic-with-zeek-in-podman.md).

## Configuration
- Live capture runs with host networking and two capabilities (`NET_RAW`, `NET_ADMIN`). `-C` ignores checksums, because packets the VM sends can have unfilled checksums.
- Logs land in the folder it starts from and are written when Zeek exits. Root owns them afterward, so I `chown` them back.
- Logs are tab-separated. I read them with `zeek-cut` inside the container, and the field names have to be exact.
- Timestamps are in UTC.

## Dependencies
- podman, from the Kali repo.
- Internet access to pull the image, which [vm-lockdown](vm-lockdown.md) still allows.

## Known Issues
- Kali's own `zeek` package cannot be installed on the current release. Forcing it or downgrading `libc6` is the wrong fix.
- `latest` is a moving tag, so record the version when it matters.
- `-w` needs a leading slash (`-w /logs`), or podman says the workdir does not exist.
- A wrong field name in `zeek-cut` gives a blank column, not an error.
- It only sees the VM's own traffic on the NAT network.
- UTC timestamps are hours ahead of local time. The same kind of mismatch broke the Wazuh syslog relay from Metasploitable.

## Related Notes
- [Suricata](suricata.md)
- [vm-lockdown](vm-lockdown.md)
- [Capture traffic with Zeek in a podman container](../runbooks/capture-traffic-with-zeek-in-podman.md)

## Tags
#service
