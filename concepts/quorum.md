# Quorum

## What It Is
Quorum is the rule that a cluster only acts when it can see more than half of its votes. Each node normally has one vote. If a node or group of nodes cannot count a majority, it assumes it might be the one that got cut off and stops making changes. This keeps two halves of a split cluster from both carrying on and disagreeing about the truth.

## How It Works
The majority is more than half of the total votes, so it is `floor(votes / 2) + 1`.

| Votes | Quorum | Can lose |
|-------|--------|----------|
| 2 | 2 | 0 nodes |
| 3 | 2 | 1 node |
| 4 | 3 | 1 node |
| 5 | 3 | 2 nodes |

Two nodes is the bad case. Quorum is 2, so if either node goes down, the survivor has 1 vote out of 2, which is not a majority. It cannot tell "the other node is dead" from "the network between us is broken", so it stops.

On Proxmox, a node without quorum makes `/etc/pve` read only. Guests that are already running keep running, but I cannot start or change guests or edit cluster config until quorum returns. It is not a crash, it is a freeze to protect the config.

The usual answers:

- A third full node. Expensive.
- A QDevice: a small external machine that runs `corosync-qnetd` and casts one tie-breaking vote for the nodes. It does not run guests and does not need to be a Proxmox node. With 2 nodes and a QDevice there are 3 votes and a quorum of 2. See [QDevice](../services/qdevice.md).
- `pvecm expected 1` tells a lone surviving node to lower its expected votes so it can work again. This is an emergency override for one node while the other is down. I should not leave it set, and I should never run it on both halves of a real split.

## In My Setup
My cluster has two nodes. Before the QDevice, `pvecm status` showed Expected votes 2 and Quorum 2, which meant a reboot of either node would freeze the other. With a VPS as the QDevice it shows Expected votes 3, Total votes 3 and Quorum 2, and I confirmed that rebooting the second node alone left the first one quorate.

The extra vote does not protect me from losing the VPS and one node at the same moment. Then there is 1 vote of 3 and the survivor freezes.

## Related Notes
- [QDevice](../services/qdevice.md)
- [Join a second Proxmox node and add a QDevice](../runbooks/join-a-second-proxmox-node-and-add-a-qdevice.md)
- [Proxmox](../services/proxmox.md)
