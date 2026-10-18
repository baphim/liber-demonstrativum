---
tags:
  - runbook
  - "#wazuh"
  - "#arch"
---

# Verify a rootcheck trojan alert

Check whether a Wazuh rule 510 "Trojaned version of file" alert is real or a false positive. The commands assume an Arch-based host that uses `pacman`. First used on the workstation, see [Wazuh alert review](../incidents/wazuh-alert-review.md).

## Step 1: List what was flagged
On the Wazuh server. The same binary can show up under several paths.
```
grep -o "Trojaned version of file '[^']*'" /var/ossec/logs/alerts/alerts.json | sort | uniq -c
```

## Step 2: Check whether the paths are symlinks to one place
On the flagged host. On Arch, `/bin`, `/sbin` and `/usr/sbin` all point into `/usr/bin`, so one binary is scanned several times.
```
ls -ld /bin /sbin /usr/sbin
readlink -f /usr/bin/<binary>
```

## Step 3: Find the owning package
If no package owns the file, treat the alert as real.
```
pacman -Qo /usr/bin/<binary>
```

## Step 4: Verify the package against its checksums
A mismatch means treat the alert as real.
```
pacman -Qkk <package>
```

## Step 5: Check why it is installed
```
pacman -Qi <package> | grep -E "Install Reason|Required By"
```

## Step 6: Remove it if nothing needs it
That ends the alert with no suppression rule to maintain.
```
sudo pacman -Rns <package>
```

## Verification
- Step 4 ends with `0 altered files`.
- `pacman -Qkk` compares only against the local package database, so it would not catch a compromise that also rewrote the database. Nothing else pointed that way here, but it is the limit of this check.
- After the next rootcheck scan, count the alerts on the Wazuh server and confirm the number has not grown.
```
grep -c "Trojaned version" /var/ossec/logs/alerts/alerts.json
```

## Related Notes
- [Wazuh alert review](../incidents/wazuh-alert-review.md)
- [Run Wazuh alert triage with a local model](run-wazuh-alert-triage.md)
- [Wazuh](../services/wazuh.md)

## Tags
#runbook
