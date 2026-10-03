---
tags:
  - runbook
  - "#wazuh"
---

# Fix Wazuh vulnerability-detector disk bloat

Known upstream Wazuh bug, likely to recur. Full incident writeup: [Wazuh dashboard error, disk full, vulnerability detector bloat](../incidents/wazuh-disk-full.md). See [Wazuh](../services/wazuh.md) for the service note.

## When to use this
Dashboard errors like `Error Pattern Handler (getPatternList)`, or any general Wazuh flakiness, combined with the disk being near or at full. Confirm before proceeding:
```
df -h /
du -h -d 2 /var/ossec/queue 2>/dev/null | sort -rh | head -5
```
If `/var/ossec/queue/vd/feed` and `/var/ossec/queue/vd_updater/tmp` are the large entries, this is the same bug.

## Step 1: Stop wazuh-manager the right way
**Do not use `systemctl stop wazuh-manager`.** Wazuh manages its own daemons via `wazuh-control`, not through reliable systemd process tracking. `systemctl stop` can report success while the real daemons keep running underneath. Use Wazuh's own tool:
```
sudo /var/ossec/bin/wazuh-control stop
```
If this errors with "Another instance is locking this process," that's `wazuh-control` correctly detecting real running daemons (check `ps aux | grep wazuh-` to confirm), not a stale lock to just delete. If daemons truly are still running despite a stop attempt, force them down directly:
```
sudo pkill -9 -f /var/ossec/bin/wazuh-
sudo pkill -9 -f wazuh_apid.py
ps aux | grep wazuh-        # should show nothing but this grep itself
```
Then clean up stale PID files and the lock before proceeding:
```
sudo rm -f /var/ossec/var/run/*.pid
sudo rm -f /var/ossec/var/start-script-lock
```

## Step 2: Clear the bloated directories, correctly
These are regenerable caches (the vulnerability feed re-downloads on next start), safe to clear. **But use the right form of the command**:
```
sudo bash -c 'rm -rf /var/ossec/queue/vd_updater/tmp/*'
sudo bash -c 'rm -rf /var/ossec/queue/vd/feed/*'
```
**Do not use `sudo rm -rf /path/*` directly.** The `*` glob expands in the *calling shell*, before `sudo` takes effect. If the calling user lacks read access into that directory (likely, these are root/`wazuh`-group only), the glob silently expands to nothing, and `rm -f -- '*'` does nothing while producing no error. This has already happened once, nothing was actually deleted, despite the command appearing to run cleanly. Wrapping the whole glob inside `sudo bash -c '...'` forces the expansion to happen with root's permissions instead.

## Step 3: Verify before restarting
```
sudo ls -la /var/ossec/queue/vd/feed    # should show only . and ..
df -h /
```
Confirm real space is back before proceeding, don't just trust that the delete commands ran without error.

## Step 4: Start back up
```
sudo /var/ossec/bin/wazuh-control start
```
Watch for "Process N not used by Wazuh, removing..." lines, this is normal if Step 1 required a force-kill, `wazuh-control` cleaning up stale references on the way up. The vulnerability feed rebuilds automatically after start, give it a few minutes before expecting the dashboard to be fully happy.

## Notes / gotchas
- This is a known upstream bug, not something specific to this lab's setup. It may recur. Consider checking `/var/ossec/queue/vd/feed` size periodically rather than waiting for disk-full again.
- Both command gotchas here (wrong stop method, unsafe glob expansion) are worth remembering more broadly, not just for this specific fix.
- If disk space stays tight even after this fix, consider whether the disk allocation needs a genuine resize rather than repeating this cleanup indefinitely.

## Related Notes
- [Wazuh](../services/wazuh.md)
- [Wazuh dashboard error, disk full, vulnerability detector bloat](../incidents/wazuh-disk-full.md)
