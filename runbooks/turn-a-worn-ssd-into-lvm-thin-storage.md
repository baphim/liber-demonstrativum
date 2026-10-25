---
tags:
  - runbook
  - "#proxmox"
  - "#storage"
---

# Turn a worn SSD into LVM-thin storage

Based on wiping a used 256 GB SSD that came with a second-hand desktop and making it guest storage on my second Proxmox node. Background on why I do not trust it: [SSD wear and SMART](../concepts/ssd-wear-and-smart.md). Values in angle brackets are placeholders.

## Why LVM-thin and not ZFS
The drive is worn and the node has 16 GB of RAM. ZFS adds write overhead and uses RAM. LVM-thin still gives me snapshots. The boot drive is a separate device, so if this one dies I lose guests and not the host.

## Step 1: Save the SMART output first
```
smartctl -a /dev/<disk> | tee /root/<disk>-smart-<date>.txt
```
This also confirms you are about to wipe the right drive, by model and serial. Read the counters before trusting them. Mine passed its self-test with 0 reallocated sectors, but the written total and the power-on hours did not fit each other.

## Step 2: Wipe it
```
wipefs -a /dev/<disk>[0-9]* /dev/<disk> && sgdisk --zap-all /dev/<disk> && partprobe /dev/<disk>
```
My first version of this block had a guard line (`... || { echo ...; false; }`) that printed a warning and did not stop anything after it when pasted into a terminal. A chain of `&&` does stop. Check the model from Step 1 before you run it either way.

## Step 3: Create the volume group and thin pool
```
pvcreate /dev/<disk> && vgcreate <vg> /dev/<disk> && lvcreate -l 95%FREE --thinpool thinpool <vg>
```
I left 5% free for thin pool metadata.

## Step 4: Add it to Proxmox, restricted to this node
```
pvesm add lvmthin <vg> --vgname <vg> --thinpool thinpool --content rootdir,images --nodes <node>
pvesm status
```
`--nodes` matters in a cluster. The storage list is shared, and an unrestricted storage shows up as broken on the other node. `pvesm status` should show it active, with the other nodes' storages `disabled` here.

## Step 5: Clean up stale boot entries and test a reboot
The old OS install left a boot entry pointing at a partition that no longer exists. `efibootmgr -v` shows the order. Mine already had the Proxmox entry first, so I only removed the stale one:
```
efibootmgr -b <entry> -B
reboot
```
After the reboot, check `/boot/efi` is on the boot drive, the cluster still has its votes, and the new storage is still active.

## Not done
Backups for guests on this storage. The backup job and its storage live on the first node, so nothing on this drive is covered. That has to happen before anything real goes on it.

## Related Notes
- [SSD wear and SMART](../concepts/ssd-wear-and-smart.md)
- [Proxmox](../services/proxmox.md)
