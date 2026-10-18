---
date: 2026-10-17
status: Open
tags:
  - incident
  - "#wazuh"
  - "#llm"
---

# Wazuh alert review: a local model flagged four alert groups

## What Happened
I built a script that pulls recent alerts from the Wazuh server and has a local model explain them. The first real run flagged four groups of alerts, so what was meant as a test turned into a review of each one. Three of the four were benign or background noise. The fourth, on the Kali VM, is still unchecked, which is why this stays open.

## Investigation
1. The script flagged four groups at level 7 or higher: rule 5712 (SSH brute force) on the VPS, rule 510 (rootcheck, trojaned `/bin/mail`) on the workstation, and rule 533 (listening ports changed) on the VPS and on Kali. The model's explanations were alarmist. It told me to isolate the workstation, which the evidence did not support.
2. **Rule 510.** I grepped `alerts.json` on the Wazuh server for the flagged paths. It listed `/bin/mail`, `/sbin/mail`, `/usr/bin/mail` and `/usr/sbin/mail`. `ls -ld /bin /sbin /usr/sbin` showed all three directories are symlinks into `/usr/bin`, so it was one binary scanned four times. `pacman -Qo` showed it belongs to `s-nail`, and `pacman -Qkk s-nail` reported 0 altered files out of 16. Nothing required the package.
3. **Rule 5712.** `sshd -T` on the VPS showed root login and password login both off. My first `journalctl | grep Accepted` returned nothing because my user is not in the `adm` or `systemd-journal` groups. Rerun with `sudo`, it showed one `Accepted publickey` line, which was me. The grep also matched `PubkeyAcceptedAlgorithms` on an unrelated line, a client offering a signature type the server rejects. `ss -tulpn` showed sshd listening on two ports. My session was on the non-default one, and a scanner was holding a connection open on the default one.
4. **Rule 533 on the VPS.** I diffed `full_log` against `previous_output` for the four alerts with a short Python snippet. Each one was `tailscaled` opening two extra UDP listeners on high ports and closing them about six minutes later.
5. **Rule 533 on Kali.** Not checked yet. The sample showed `charon` and `rsyslogd` listening, which I did not expect on a stock Kali.

Problems getting the script running:
- `ssh` to the Wazuh server asked for a password because my key has a custom name and ssh only offers default key names. I already had a `Host` entry in `~/.ssh/config` pointing at the key, so the fix was using the alias.
- I pasted only the run line and skipped the step that creates the script, so Python could not find the file.

## Root Cause
There was no single cause. Each alert had its own explanation:
- **Rule 510:** false positive. Rootcheck matches strings that appear inside a legitimate mail client, and the symlinks made it fire four times.
- **Rule 5712:** internet background noise against a server with key-only auth, so none of it can succeed.
- **Rule 533 on the VPS:** normal `tailscaled` behavior.
- **Rule 533 on Kali:** unknown.

The model made two of these sound worse than they were. The 200-character alert sample hid what changed in the ports alert, so both models I tried blamed the reverse proxy. The input was the weak point, not the model, see [Local models](../services/local-models.md).

## Fix
- Removed the unused `s-nail` package from the workstation. No suppression rule was needed.
- No change for the SSH alerts or the VPS port alerts yet.

## Still Open
- Check the Kali ports.
- Decide whether to close the default SSH port on the VPS, after checking what else uses it and the provider's cloud firewall.
- Consider fail2ban, or binding sshd to the tailnet address only with the provider's console as the fallback.
- Rule 5712 is level 10 for brute force against a key-only server, which is too loud. A local rule can lower it for that host once the port decision is made.
- Filter the ephemeral `tailscaled` ports in the agent's netstat command in `ossec.conf`, not with a rule. A rule matching `tailscaled` would hide real port changes.
- Improve the script: include the ports diff, distinct paths for rule 510, and a one-line note per rule about the usual cause and first check.
- Restrict the workstation's key on the Wazuh server to a forced read-only command, so a stolen key can only read alerts.
- Confirm the rootcheck alerts stop after the next scan.

## Lessons
- Detection belongs in code. The model should explain what the code found, and its answer gets checked against the raw alert.
- A scary recommendation is a prompt to verify, not to act.
- An empty `journalctl` result may be a permissions problem.

## Related Notes
- [Wazuh](../services/wazuh.md)
- [Ollama](../services/ollama.md)
- [Run Wazuh alert triage with a local model](../runbooks/run-wazuh-alert-triage.md)
- [Verify a rootcheck trojan alert](../runbooks/verify-a-rootcheck-trojan-alert.md)
- [Check a VPS for SSH exposure](../runbooks/check-vps-ssh-exposure.md)

## Tags
#incident
