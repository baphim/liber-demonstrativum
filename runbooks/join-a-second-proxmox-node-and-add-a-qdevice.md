---
tags:
  - runbook
  - "#proxmox"
  - "#cluster"
  - "#tailscale"
---

# Join a second Proxmox node and add a QDevice

Based on adding a used desktop as the second node of my Proxmox cluster and giving the pair a third vote from a VPS. Two nodes alone freeze each other when one goes down, see [Quorum](../concepts/quorum.md). The result is in [Proxmox](../services/proxmox.md) and [QDevice](../services/qdevice.md). Values in angle brackets are placeholders.

## Order
1. Prepare the new node.
2. Lock down the tailnet policy before any Proxmox node joins it.
3. Firewall the QDevice host.
4. Create the cluster and join.
5. Fix node-to-node SSH and the storage definitions.
6. Add the QDevice.
7. Test failure.

## Step 1: Prepare the new node
- Match the Proxmox version on the existing node. `pveversion` on both. The installer shipped an older point release, so I switched to the no-subscription repo and ran a full upgrade until the versions matched.
- The installer ignored my `maxroot` setting and root came out at 12 GB. `vgs` showed free space in the volume group, so I fixed it after the install:
  ```
  lvextend -r -l +100%FREE /dev/pve/root
  ```
  The `-r` resizes the filesystem in the same step.
- Set up key-only SSH for a normal user, the same model as the existing node.
- Check the clock with `timedatectl`. Skewed time breaks things downstream.
- Take a `vzdump` of the existing node's guests and a `tar` of `/etc/pve` before joining anything.

## Step 2: Lock down the tailnet first
The Proxmox nodes are the most important machines in the lab, and a QDevice over the tailnet means they have to join it. I did not want the rest of the tailnet to be able to reach them, so I changed the policy before joining.

1. Replace the default allow-all policy with explicit `grants`, and add a `tests` section.
2. Add a tag for the nodes (`tag:pve`) that may reach only the QDevice host, on the qnetd port (TCP 5403) and, temporarily, SSH.
3. Join each node with `tailscale up --advertise-tags=tag:pve --shields-up --accept-dns=false`, then disable key expiry in the admin console.
4. Check from a phone with wifi off that the tailnet services I use still load.
5. Verify a TCP connection from the node to the QDevice host. If `nc` is missing, `timeout 3 bash -c '</dev/tcp/<host>/<port>'` works.

Why each flag matters: [Tailscale tags and grants](../concepts/tailscale-tags-and-grants.md).

## Step 3: Firewall the QDevice host
Before exposing a new service on the VPS, I checked its provider console. The cloud firewall list was empty, so the host was relying on nothing outside the VM. I created one that allows only the ports the VPS actually serves, and does not allow SSH on the default port. Before attaching it, I confirmed how I really reach the box (a non-default SSH port), so I did not lock myself out. The provider's firewall is stateful, so replies are fine.

## Step 4: Create the cluster and join
On the existing node:
```
pvecm create <cluster-name>
```
On the new node:
```
pvecm add <existing-node-address>
```
My first `pvecm add` failed with `invalid corosync.conf, no nodes found`, because I had never run `pvecm create`. Nothing was damaged. Then `pvecm status` and `pvecm nodes` on either node should show both nodes and `Quorate`.

## Step 5: Fix SSH and storage
**SSH between nodes.** `ssh <other-node>` hung, then failed with `Host key verification failed`, then the other side could not resolve the first node's name. Neither node had the other in `/etc/hosts`. The fix:
1. Add each node's name and address to `/etc/hosts` on the other.
2. `pvecm updatecerts` on both.
3. Run an interactive `ssh` in each direction, check the fingerprint and accept it.

I kept losing track of which commands ran on which node. I now write the node name in front of every command in my notes.

**Storage.** The cluster shares `/etc/pve/storage.cfg`, so the first node's storages showed up on the new node, where they do not exist. Restrict each one:
```
pvesm set <storage> --nodes <first-node>
```

## Step 6: Add the QDevice
- Install `corosync-qnetd` on the QDevice host and `corosync-qdevice` on both nodes.
- Bind qnetd to the tailnet address only, and make it wait for `tailscale0` at boot. The config is in [QDevice](../services/qdevice.md).
- `pvecm qdevice setup <tailnet-address>` connects to the qnetd host as root over SSH. I allowed root login for that moment only and turned it off after. A later `sshd -T` showed `permitrootlogin no`.
- At one point I told myself I had run the setup when I had not. `pvecm status` is the check, not memory.

Expected result: Expected votes 3, Total votes 3, Quorum 2, flags `Quorate` and `Qdevice`.

## Step 7: Test failure
- Reboot the second node alone. The first should stay quorate.
- Reboot the QDevice host. qnetd should come back by itself, the cluster should keep 2 of 3 votes while it is down, and the QDevice should reconnect.
- Remove SSH from the grant once the setup works, and confirm the nodes still reach the qnetd port.

## What this does not do
The cluster shares config and the web UI. It does not share shell config or SSH logins, and I have not set up shared storage or HA, so guests do not move between nodes by themselves. Losing the QDevice host and one node together still freezes the survivor.

## Related Notes
- [Quorum](../concepts/quorum.md)
- [QDevice](../services/qdevice.md)
- [Tailscale tags and grants](../concepts/tailscale-tags-and-grants.md)
- [Check a VPS for SSH exposure](check-vps-ssh-exposure.md)
