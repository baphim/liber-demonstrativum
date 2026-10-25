# SSD wear and SMART

## What It Is
Flash cells in an SSD wear out after a limited number of writes. Manufacturers rate that as TBW (terabytes written): the amount of data the drive is expected to take before it can no longer be trusted. SMART is the drive's self reporting. Tools such as `smartctl` and CrystalDiskInfo read it and show host writes, remaining life, reallocated sectors and similar counters.

## How It Works
- "Total host writes" is how much the computer has written. A drive near or past its TBW rating is living on borrowed time, even if it still reports as healthy.
- The health percentage is computed by the drive's firmware from its own counters, and vendors differ. Some counters are normalized values and some are raw. Two counters that disagree, or a raw value that does not fit the drive's age, mean I should not trust the summary.
- Consumer 256 GB drives have low TBW ratings compared with larger ones, so a very high figure on a small drive is a bigger deal than the same figure on a 1 TB drive. I did not find the rated TBW for this model, so I cannot say how far past it the drive is.
- Wear is about writes, not power on hours. A drive that mostly sat idle can have a long life left, and one that lived under a database can be done.

## Reading one in practice
I ran `smartctl -a` on a used 256 GB SSD that came with a second-hand desktop, before wiping it. It passes its self-test, has 0 reallocated sectors, and shows a Media Wearout Indicator of 100. `Total_LBAs_Written` is a raw 126768, which matches the roughly 126 TB CrystalDiskInfo showed if the unit is GB. `Power_On_Hours` is only 134. That does not fit: 126 TB in 134 hours would be about 900 GB every hour. Either the hours counter was reset or I have the write unit wrong, and I cannot tell which.

So the drive reports itself as healthy, and its wear history is unknown, which is a different thing from healthy. I saved the full `smartctl` output before wiping, so I can compare later.

## In My Setup
I treat that drive as unreliable, not as dead. It became guest storage on the second node, and the boot drive is a separate Optane module, so a failure of the SSD would take out the guests and not the host. I chose LVM-thin over ZFS for it, because ZFS adds write overhead and uses RAM. See [Turn a worn SSD into LVM-thin storage](../runbooks/turn-a-worn-ssd-into-lvm-thin-storage.md).

I will replace the drive when I can afford a better one. Backups for guests on that node are not set up yet, and that has to happen before anything real goes on it.

## Related Notes
- [Turn a worn SSD into LVM-thin storage](../runbooks/turn-a-worn-ssd-into-lvm-thin-storage.md)
- [Proxmox](../services/proxmox.md)
