---
tags:
  - runbook
  - "#wazuh"
---

# Upgrade Wazuh central components and agents

Upgrade an all-in-one Wazuh stack (indexer, manager, Filebeat, dashboard) to a new patch release, then bring every agent up to the same version without ever letting an agent get ahead of the manager. Validated on a patch upgrade, RHEL-family manager, Debian-family agents, one Arch-family workstation agent. For a minor or major version jump, read the official upgrade guide for that version first, since the extra configuration steps it lists depend on the version you start from. See [Wazuh](../services/wazuh.md) for the service note.

Rule that drives the whole order: the manager must always be the same version or newer than every agent. Central components must all match to the patch level.

## Step 1: Pick the target and check free disk
Check the current version and confirm the manager host has comfortable headroom before starting. The vulnerability scanner decompresses a multi-gigabyte feed snapshot into a temp directory on the first manager start after an upgrade, on top of an already large feed directory. A disk that is close to full before the upgrade can run out of space mid-upgrade. See [Fix Wazuh vulnerability-detector disk bloat](fix-wazuh-disk-bloat.md) if that happens.

## Step 2: Snapshot and back up
Take a VM snapshot of the manager host before touching anything. The indexer's own upgrade path does not support a downgrade past a certain version, so the snapshot is the real rollback. Back up `/var/ossec/etc` and `/var/ossec/integrations` (this covers the main config, custom rules and decoders, and any custom integration scripts and their credential files). In the dashboard, export saved objects.

## Step 3: Enable the manager's package repo
The repo is normally disabled between upgrades so nothing updates unattended. Re-enable it and confirm the new version is visible to the package manager before proceeding.

## Step 4: Stop Filebeat and the dashboard
Both have to be down before the indexer and manager upgrade.

## Step 5: Back up the indexer security configuration
The indexer ships a tool for this that prompts for the indexer admin password. Run it before changing anything.

## Step 6: Restrict shard allocation and flush
Set cluster shard allocation to primaries only and flush, so the upgrade does not have to contend with shard relocation while nodes restart.

## Step 7: Upgrade the indexer
Stop the manager and the indexer, upgrade the indexer package, then start it again. If the package manager offers to replace a tuning file such as JVM heap options, diff the old and new versions and reapply any custom settings.

## Step 8: Indexer post-upgrade actions
Rerun the security init tool, confirm the node list, re-enable normal shard allocation, and check cluster health settles on green with no unassigned shards. A brief yellow right after restart is normal; recheck after a short wait.

## Step 9: Upgrade the manager
Upgrade the manager package and start it. A modified main config file is not replaced on upgrade, so custom integrations and rules survive.

## Step 10: Check the manager and watch the vulnerability scanner
Confirm the new version, confirm any custom integrations re-enabled in the log, confirm their files kept their original ownership and permissions, and watch free disk while the vulnerability scanner does its first-run decompression. If the log shows the scanner running out of space, stop before touching Filebeat or the dashboard, grow the disk, then restart the manager and let the scanner retry.

## Step 11: Filebeat
Refresh the Wazuh Filebeat module files and template for the new version, upgrade the package (this usually changes nothing if Filebeat itself is pinned to a compatible version), then verify pipelines, index templates and output all pass their setup and test commands.

## Step 12: Dashboard
Upgrade the dashboard package, diff its config file against the pre-upgrade copy and reapply any custom settings that were reset, confirm its TLS certificate paths still match what is on disk, and confirm its plugin list shows nothing outdated.

## Step 13: Verify the stack
Confirm every central component reports the same new version. Log in to the dashboard and confirm agents, saved dashboards and recent alerts are all present. Retest any alert integration with a fake alert to confirm the whole chain still works end to end.

## Step 14: Lock the manager repo again
Disable the repo so nothing upgrades unattended until the next planned upgrade. A version lock on the three central packages is an optional extra layer. Keep the pre-upgrade snapshot for a day or two, then remove it once you are confident, since an old snapshot that no longer matches the current disk size will also shrink the disk if you ever roll back to it.

## Step 15: Find which agents are behind
The manager has a built-in command that lists agents older than itself. It is worth also checking every agent's real version directly, which catches the opposite and unsupported case: an agent that ended up newer than the manager because it was not pinned and a routine system update pulled a newer agent package before the manager caught up.

## Step 16: Upgrade the package-manager-based agents, one canary first
On each agent host: unhold the package if it is pinned, update and upgrade just that package, then re-hold it. Start with a low-stakes host and do any internet-facing host last and alone. Confirm each one from the manager afterward: new version, and reporting active.

## Step 17: Agents on a different package manager, and new installs
A workstation on a rolling-release distro needs its own equivalent hold mechanism checked and temporarily lifted for the upgrade, then restored. An offline agent gets upgraded and pinned the next time it is powered on. For installing a brand new agent, do it only after the manager upgrade, and point the agent at the manager by a reserved address rather than a name that depends on infrastructure the agent should not need (such as internal DNS), for hosts where that dependency would be a problem.

## Verification
- The manager's own "agents behind" command shows nothing.
- Every agent reports active (except one that is intentionally kept powered off between uses) and on the same version as the manager.
- The pin or hold is confirmed on every agent host.
- The dashboard loads with no update banner and alerts are indexing.
- Any alert integration's end-to-end test still arrives.
- Free disk on the manager host stays comfortably under control after the vulnerability scanner finishes its first run.

## Related Notes
- [Wazuh](../services/wazuh.md)
- [Fix Wazuh vulnerability-detector disk bloat](fix-wazuh-disk-bloat.md)
- [Wazuh dashboard error, disk full, vulnerability detector bloat](../incidents/wazuh-disk-full.md)

## Tags
#runbook
