---
date: 2026-09-19
status: Resolved
tags:
  - incident
  - "#wazuh"
---

# Wazuh dashboard error, disk full, vulnerability detector bloat

## What Happened
The Wazuh dashboard was throwing `Error Pattern Handler (getPatternList)` from `WzMenu.loadIndexPatternsList`. Root cause turned out to be a disk-full condition, itself caused by a known Wazuh bug, plus two separate operational gotchas hit while trying to fix it.

## Investigation
- `df -h` showed the root filesystem at 100% capacity, nearly no free space. A full disk on the indexer/manager causes exactly this class of dashboard failure (saved-object/index-pattern lookups failing), so this was almost certainly the real cause, not a dashboard bug itself.
- Narrowed it to the vulnerability-detector module's feed directory and a staging temp directory under it.
- Confirmed via research this is a known upstream Wazuh bug (GitHub issue #32854): the Vulnerability Detector module accumulates temp and feed data without cleaning up. Official fix: stop wazuh-manager, clear both directories, start it back up, they're regenerable caches, not real data.

## Root Cause(s)
Three separate problems stacked on top of each other:

1. **The known Wazuh bug** (above), the actual reason disk filled up in the first place.
2. **`systemctl stop wazuh-manager` doesn't reliably stop Wazuh.** Wazuh manages its own daemons via `wazuh-control`, not directly through systemd process tracking. `systemctl stop` reported success, but every real daemon was still running underneath, same PIDs, same original start time from that morning's boot. Every subsequent start attempt was trying to start a second set of daemons on top of the still-running first set, which caused the actual startup failures and a stale-looking lock file (which was not stale at all, `wazuh-control` was correctly detecting the real running daemons and refusing to start over them).
3. **`sudo rm -rf /path/*` silently did nothing.** The `*` glob gets expanded by the calling user's shell before `sudo` ever runs. The calling account didn't have read access into the target directory, so the glob expanded to nothing, and the delete failed silently rather than with an error. Disk usage barely moved, and every original file was still there afterward. Confirmed via `lsof +L1` that nothing was holding deleted-but-open file handles either, the delete had genuinely never run. Fix: wrap the whole glob in a root shell so the expansion itself happens with root's permissions.

## Fix
1. Force-killed the real running daemons directly, since the normal stop command wouldn't proceed given the (correct) belief that an instance was still running.
2. Cleared stale PID files and the lock.
3. Cleared the actual bloated directories, correctly this time, wrapped in a root shell. Confirmed empty, and confirmed real space back via `df -h`.
4. Started wazuh-manager back up. Came up clean, flagging and cleaning up the stale PID references from the earlier stuck instance on the way up.
5. The feed rebuilt automatically on restart. Dashboard confirmed working afterward.

## Lessons for next time
- **Always use `wazuh-control`, not `systemctl`, to stop/start Wazuh manager.** `systemctl` doesn't reliably reflect or control the real daemon state.
- **Never trust a bare `sudo rm -rf /path/*` on a directory the calling user can't read into.** Either wrap the whole glob in a root shell, or use `find /path -mindepth 1 -delete` under sudo, which doesn't rely on shell-side glob expansion at all.
- This is a known upstream issue that may recur. Worth checking the feed directory's size periodically rather than waiting for the disk to actually fill again.
- The disk has very little margin once this feed grows back. Worth deciding whether to resize it as a buffer on top of watching for the bloat itself.

## Related Notes
- [Wazuh](../services/wazuh.md)
- [Fix Wazuh vulnerability-detector disk bloat](../runbooks/fix-wazuh-disk-bloat.md)

## Tags
#incident
