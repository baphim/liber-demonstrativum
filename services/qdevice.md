---
status: Running
host: "VPS"
reviewed: 2026-10-25
tags: [service]
---

# QDevice

## Overview
| Field | Value |
|-------|-------|
| **Purpose** | Third vote for the two-node Proxmox cluster, so it stays quorate when either node is down |
| **Host** | The VPS runs the `corosync-qnetd` server. Both Proxmox nodes run the `corosync-qdevice` client. |
| **Status** | Running |

This is temporary. I want a Raspberry Pi on the LAN to do this job. I do not know when I can get one, and prices went up this year, so the VPS is the stand-in. See [Quorum](../concepts/quorum.md) for why a third vote is needed.

## Access
Nothing listens on the VPS's public address for this. The cloud firewall does not open the port, and `corosync-qnetd` is bound to the VPS's tailnet address anyway. Over the tailnet, only nodes with the `tag:pve` tag may reach it, and only on that one port (TCP 5403). See [Tailscale tags and grants](../concepts/tailscale-tags-and-grants.md).

## Deployment
`apt install corosync-qnetd` on the VPS and `apt install corosync-qdevice` on both nodes. Then, on one node, `pvecm qdevice setup <tailnet-address>`. That command connects to the qnetd host as root over SSH, so root login had to be allowed on the VPS for the moment it ran. I turned the exception back off afterward and confirmed `permitrootlogin no` with `sshd -T`.

`pvecm status` afterward showed Expected votes 3, Total votes 3, Quorum 2, and the flags `Quorate` and `Qdevice`. The full steps are in [Join a second Proxmox node and add a QDevice](../runbooks/join-a-second-proxmox-node-and-add-a-qdevice.md).

## Configuration
qnetd is bound to the tailnet address so it never listens publicly. In `/etc/default/corosync-qnetd`:

```
COROSYNC_QNETD_OPTIONS="-l <tailnet-address>"
```

A systemd drop-in at `/etc/systemd/system/corosync-qnetd.service.d/tailnet.conf` makes the service wait for Tailscale. Without it, qnetd could start before `tailscale0` has its address and fail to bind:

```
[Unit]
After=tailscaled.service
Wants=tailscaled.service

[Service]
Restart=on-failure
RestartSec=5
ExecStartPre=/bin/sh -c 'for i in $(seq 1 60); do ip -4 addr show tailscale0 | grep -q <tailnet-address> && exit 0; sleep 1; done; exit 1'
```

Check it with `pvecm status` on either node, and `ss -tlnp | grep 5403` on the VPS.

Verified:
- With the second node rebooted alone, the first stayed quorate.
- `corosync-qdevice` is enabled on both nodes.
- I rebooted the VPS. `corosync-qnetd` came back on its own, bound to the tailnet address, and the QDevice reconnected. While it was down the cluster had 2 of 3 votes and stayed quorate.
- I removed SSH from the `tag:pve` grant after setup. The nodes can reach the VPS on 5403 only.

## Dependencies
- [Tailscale](tailscale.md), since the nodes reach the VPS over the tailnet
- [Proxmox](proxmox.md) and corosync on both nodes

## Known Issues
- Losing the VPS and one node at the same time freezes the cluster. That is the cost of using a remote box.
- Both Proxmox nodes are on the tailnet only because of this. Removing that exposure is the reason to move to a Pi.
- Swapping to a Pi later: `pvecm qdevice remove`, then `pvecm qdevice setup <Pi LAN address>`, then take both nodes off the tailnet. If `remove` needs to reach the VPS and cannot, SSH goes back into the grant for that step only.

## Related Notes
- [Quorum](../concepts/quorum.md)
- [Tailscale tags and grants](../concepts/tailscale-tags-and-grants.md)
- [Join a second Proxmox node and add a QDevice](../runbooks/join-a-second-proxmox-node-and-add-a-qdevice.md)

## Tags
#service
