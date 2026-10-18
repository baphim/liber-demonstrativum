---
tags:
  - runbook
  - "#wazuh"
  - "#ssh"
---

# Check a VPS for SSH exposure

Decide whether a Wazuh rule 5712 alert (SSH brute force, non-existent user) on an internet-facing server is background noise or a real problem. If password login is off, guessing cannot succeed and the alert is noise. First used in the [Wazuh alert review](../incidents/wazuh-alert-review.md).

## Step 1: Confirm password and root login are off
```
sudo sshd -T | grep -Ei '^(passwordauthentication|permitrootlogin|kbdinteractiveauthentication|pubkeyauthentication|authenticationmethods)'
```

## Step 2: List every port sshd listens on, and where each is set
```
sudo sshd -T | grep -i '^port'
sudo grep -rn '^Port' /etc/ssh/sshd_config /etc/ssh/sshd_config.d/
```

## Step 3: Check everything that listens
Only SSH, the web server and Tailscale should be there.
```
sudo ss -tulpn
```

## Step 4: Review successful logins over the last week
Without `sudo`, `journalctl` hides system messages from a user who is not in `adm` or `systemd-journal`, so an empty result can be a permissions problem and not a clean bill. Match on `Accepted publickey` so `PubkeyAcceptedAlgorithms` lines do not show up.
```
sudo journalctl -u ssh --no-pager --since "7 days ago" | grep "Accepted publickey" | tail
```

## Step 5: See who is connected now, and on which port
```
sudo ss -tn state established '( sport = :<port-a> or sport = :<port-b> )'
echo $SSH_CONNECTION
```

## Step 6: Check recent attempts
Look for anything that is not `[preauth]`.
```
sudo journalctl -u ssh --since "1 hour ago" --no-pager | tail -n 40
```

## Verification
- `passwordauthentication no` and `permitrootlogin no`, with `pubkeyauthentication yes`.
- Every `Accepted publickey` line is a user and address you recognize.
- Failed attempts come from many unrelated addresses, target `root` or made-up names, and end in `[preauth]`.
- Any `Accepted` line for another user or an unknown address is real. Stop and investigate before anything else.
- A provider-level cloud firewall sits outside the VM, so check it in the provider's console too.
- To drop a port safely: keep one session open, comment out the `Port` line, run `sudo sshd -t`, run `sudo systemctl reload ssh`, then test a fresh login on the remaining port from a second terminal before closing the first.

## Related Notes
- [Wazuh alert review](../incidents/wazuh-alert-review.md)
- [Run Wazuh alert triage with a local model](run-wazuh-alert-triage.md)
- [Wazuh](../services/wazuh.md)

## Tags
#runbook
