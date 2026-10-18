---
tags:
  - runbook
  - "#wazuh"
  - "#llm"
---

# Run Wazuh alert triage with a local model

Pull recent alerts from the Wazuh server, group them in code, and have a local model explain each group. The script takes four arguments: the SSH target, how many lines to read, the minimum rule level, and the model. The model only sees what the code flagged, see [Detection in code, explanation in the model](../concepts/detection-in-code-explanation-in-the-model.md). First real run: [Wazuh alert review](../incidents/wazuh-alert-review.md).

## Step 1: Make sure Ollama is running
```
systemctl is-active ollama
```

## Step 2: Test key-based SSH to the Wazuh server
Use the alias from `~/.ssh/config`. ssh only offers default key names such as `id_rsa` on its own, so a key with a custom name is only tried for hosts that name it in the config. `BatchMode` makes this fail instead of prompting if the key is not working.
```
ssh -o BatchMode=yes <wazuh-alias> "tail -n 1 /var/ossec/logs/alerts/alerts.json"
```
The alerts file is readable only by the `wazuh` user and group, so the SSH account must be in that group. Log out and back in after adding it.

## Step 3: Run the script
Last 2000 alerts at level 7 or higher. Use `12` as the third argument to see only High and above.
```
python3 ~/llm/wazuh_triage.py <wazuh-alias> 2000 7
```

## Step 4: Read the findings first
The FINDINGS section was produced by code. The EXPLANATION section came from the model, so check each item against the raw alert before acting on it.

## Step 5: Expand the "listening ports changed" alerts
For rule 533 the sample is cut short, and the model can only give a generic answer. To see what actually changed, run this on the Wazuh server, changing the agent name.
```
python3 - <<'PY'
import json, difflib
for l in open('/var/ossec/logs/alerts/alerts.json'):
    try: a = json.loads(l)
    except ValueError: continue
    if a.get('rule', {}).get('id') != '533' or a.get('agent', {}).get('name') != '<agent>':
        continue
    prev = a.get('previous_output', '').splitlines()
    now = a.get('full_log', '').splitlines()
    print(a['timestamp'])
    for d in difflib.unified_diff(prev, now, lineterm='', n=0):
        if d[:1] in '+-' and d[:3] not in ('+++', '---'):
            print('  ', d)
PY
```

## Verification
- FINDINGS prints, and the EXPLANATION stays on those findings without adding any of its own.
- A scary recommendation, such as isolating a machine, gets checked against the raw alert first. The model told me to isolate the workstation over a false positive.
- Alert timestamps are UTC, which is four hours ahead of local time during daylight saving.

## Related Notes
- [Ollama](../services/ollama.md)
- [Verify a rootcheck trojan alert](verify-a-rootcheck-trojan-alert.md)
- [Check a VPS for SSH exposure](check-vps-ssh-exposure.md)
- [Wazuh](../services/wazuh.md)

## Tags
#runbook
