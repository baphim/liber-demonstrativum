---
tags:
  - concept
  - "#databases"
---

# LSM-trees (RocksDB)

Log-Structured Merge-tree (LSM-tree) databases operate in a way that never modifies a file once it's created. They first write data to a buffer stored in memory. When memory is full, the data gets pushed to disk as an `.sst` (SSTable) file, a small immutable file. Usually these files undergo compaction, where they are merged into larger files and files flagged to be deleted are actually deleted.

RocksDB is a concrete implementation of an LSM-tree, and one of the most widely used ones. Other examples include LevelDB (which RocksDB was originally forked from) and Cassandra's storage engine.

## Why this matters here
In the case of [Wazuh](../services/wazuh.md)'s vulnerability-detector module, which uses RocksDB to store its feed data, there is a known upstream bug where these `.sst` files accumulate rapidly over time rather than being cleaned up by compaction as expected. This filled the feed directory to several gigabytes over about a month, eventually filling the VM's disk entirely and breaking the dashboard. The root cause was not deeply investigated (whether compaction genuinely isn't running, or something is preventing old files from being reclaimed), but the manual fix, clearing the feed and letting it rebuild, was documented for future reference. See [Fix Wazuh vulnerability-detector disk bloat](../runbooks/fix-wazuh-disk-bloat.md).

## Related Notes
- [Wazuh](../services/wazuh.md)
- [Fix Wazuh vulnerability-detector disk bloat](../runbooks/fix-wazuh-disk-bloat.md)
- [Wazuh dashboard error, disk full, vulnerability detector bloat](../incidents/wazuh-disk-full.md)

## Tags
#concept
