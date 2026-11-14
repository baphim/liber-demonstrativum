---
tags:
  - runbook
  - "#proxmox"
---

# Decommission an LXC

Remove a container that's no longer needed without leaving a ghost device on the tailnet or a live credential behind. The order matters: take the device off the tailnet before destroying the container, not after.

## Step 1: Save anything worth keeping
Check for unpushed work and copy out any editor or service configuration that isn't already in version control.
```
cd <repo>
git status
git log origin/main..HEAD --oneline
```

## Step 2: Revoke credentials
Delete any access token issued for this container, and remove its entries from the password manager. A token or stored credential stays valid after the container is gone, so this has to happen before, or it's easy to forget once the container no longer exists to remind anyone.

## Step 3: Remove it from the tailnet
From inside the container, as root:
```
tailscale serve reset
tailscale logout
```
Confirm the device is gone from the tailnet admin console afterward, and remove it there if it lingers.

## Step 4: Stop and destroy the container
```
pct stop <ctid>
pct destroy <ctid> --purge
```
`--purge` also removes it from backup jobs and any replication config. For a safer middle step on something that might still be needed, stop it and turn off start-at-boot first, then destroy it later once it's clear nothing broke.

## Step 5: Clean up references
Remove any local DNS record pointing at it, remove its entry from the monitoring system if one was enrolled, and update this vault's notes for it to say it's retired.
```
pct list
```

## Verification
- The container no longer appears in `pct list`.
- The device is gone from the tailnet admin console.
- Any issued token or stored credential for it is deleted.
- This vault's notes for it say retired.

## Related Notes
- [spyderbox](../services/spyderbox.md)
- [Set up spyderbox](set-up-spyderbox.md)

## Tags
#runbook
