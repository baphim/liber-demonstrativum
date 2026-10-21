---
tags:
  - runbook
  - "#wazuh"
  - "#ntfy"
---

# Send Wazuh alerts to ntfy

Push Wazuh alerts at level 10 and up to my phone through [ntfy](../services/ntfy.md), using a publish-only credential so a leaked token cannot read anything or touch other topics. Values in angle brackets are placeholders. Written for a Wazuh manager on a RHEL-family system, with Wazuh's bundled Python at `/var/ossec/framework/python`.

## Step 1: Check the manager can reach ntfy
The manager has to use the tailnet URL, not a LAN address. Test with a silent password prompt so nothing lands in shell history:
```
curl -u <admin-user> -d "test from wazuh manager" https://ntfy.<tailnet>.ts.net/wazuh-alerts
```
`-u <admin-user>` with no password makes curl prompt for it.

## Step 2: Publish-only user
On the ntfy host:
```
runuser -u ntfy -- ntfy user add wazuh-bot
runuser -u ntfy -- ntfy access wazuh-bot wazuh-alerts write-only
runuser -u ntfy -- ntfy token add --label="wazuh manager" wazuh-bot
runuser -u ntfy -- ntfy access
```
The token prints once, so store it in a password manager straight away. `ntfy access` should show the bot with write-only on its one topic and nothing else. Test from the manager:
```
read -rs NTFY_TOKEN
curl -H "Authorization: Bearer $NTFY_TOKEN" -d "test from wazuh token" https://ntfy.<tailnet>.ts.net/wazuh-alerts
unset NTFY_TOKEN
```

## Step 3: Store the token on the manager
The integration runs as root through `wazuh-integratord`, so the token lives in a root-owned file instead of being hardcoded in the script:
```
sudo install -m 600 -o root -g root /dev/null /var/ossec/etc/ntfy.token
sudo nano /var/ossec/etc/ntfy.token
sudo ls -l /var/ossec/etc/ntfy.token
```
The token is the only line. A 32 character token plus a newline is 33 bytes, so a different size means something extra got pasted. Then open it to the `wazuh` group, in case the integration runs as that group:
```
sudo chgrp wazuh /var/ossec/etc/ntfy.token
sudo chmod 640 /var/ossec/etc/ntfy.token
```

## Step 4: The integration script
Custom integrations must be named `custom-*`. Put two files in `/var/ossec/integrations/`: a shell wrapper that calls Wazuh's bundled Python, and the Python script.

`custom-ntfy`:
```
#!/bin/sh
exec /var/ossec/framework/python/bin/python3 /var/ossec/integrations/custom-ntfy.py "$@"
```
`custom-ntfy.py`:
```python
#!/var/ossec/framework/python/bin/python3
import json
import ssl
import sys
import urllib.request

TOKEN_FILE = "/var/ossec/etc/ntfy.token"
CA_FILE = "/etc/pki/tls/certs/ca-bundle.crt"


def main():
    alert_file = sys.argv[1]
    hook_url = sys.argv[3]

    with open(alert_file) as f:
        alert = json.load(f)
    with open(TOKEN_FILE) as f:
        token = f.read().strip()

    rule = alert.get("rule", {})
    agent = alert.get("agent", {}).get("name", "manager")
    level = int(rule.get("level", 0))
    desc = rule.get("description", "no description")

    body = (
        f"{desc}\n"
        f"Agent: {agent}\n"
        f"Rule: {rule.get('id', '?')} (level {level})\n"
        f"{alert.get('timestamp', '')}"
    )
    priority = 5 if level >= 13 else 4 if level >= 10 else 3

    req = urllib.request.Request(hook_url, data=body.encode("utf-8"), method="POST")
    req.add_header("Authorization", f"Bearer {token}")
    req.add_header("Title", f"Wazuh L{level}: {agent}")
    req.add_header("Priority", str(priority))
    req.add_header("Tags", "rotating_light")
    ctx = ssl.create_default_context(cafile=CA_FILE)
    urllib.request.urlopen(req, timeout=10, context=ctx)


if __name__ == "__main__":
    main()
```
Priority is 5 for level 13 and up, 4 for 10 to 12, and 3 for anything lower.
```
sudo chown root:wazuh /var/ossec/integrations/custom-ntfy /var/ossec/integrations/custom-ntfy.py
sudo chmod 750 /var/ossec/integrations/custom-ntfy /var/ossec/integrations/custom-ntfy.py
```

Hand test with a fake alert. `wazuh-integratord` passes the alert file, an API key and the hook URL as arguments 1 to 3, so the empty `""` stands in for the API key:
```
echo '{"rule":{"id":"99999","level":12,"description":"Manual integration test"},"agent":{"name":"test-agent"},"timestamp":"now"}' > /tmp/test-alert.json
sudo /var/ossec/integrations/custom-ntfy /tmp/test-alert.json "" https://ntfy.<tailnet>.ts.net/wazuh-alerts
rm /tmp/test-alert.json
```

> **Gotcha:** My first hand test failed with `ssl.SSLCertVerificationError: certificate verify failed: unable to get local issuer certificate`. System `curl` worked because it uses the OS cert store, but Wazuh's bundled Python has its own OpenSSL setup that does not know where the system CAs live. The fix is to point the script at the system CA bundle (the `CA_FILE` and `ssl.create_default_context(cafile=...)` lines above), not to turn verification off.

## Step 5: Wire it into ossec.conf
Back up first:
```
sudo cp -p /var/ossec/etc/ossec.conf /var/ossec/etc/ossec.conf.bak-ntfy
```
Add this at the top level of `<ossec_config>`, as a sibling of `<global>` and `<command>`:
```
<integration>
  <name>custom-ntfy</name>
  <hook_url>https://ntfy.<tailnet>.ts.net/wazuh-alerts</hook_url>
  <level>10</level>
  <alert_format>json</alert_format>
</integration>
```
`<name>` has to match the wrapper filename exactly, and `json` is required because the script parses JSON. Level 10 is a starting threshold.

> **Gotcha:** When I pasted the block for a sanity check before saving, it had two typos. I had typed `?` instead of `>` on the `<name>` line, and the tailnet hostname had an extra dot, which would have failed DNS without an obvious error.

Validate, restart and confirm:
```
sudo /var/ossec/bin/wazuh-analysisd -t
sudo systemctl restart wazuh-manager
sudo systemctl status wazuh-manager --no-pager
sudo grep -i integrat /var/ossec/logs/ossec.log | tail -5
```
The log should show `wazuh-integratord: INFO: Started` and `Enabling integration for: 'custom-ntfy'`.

## Step 6: End-to-end test
Rule 5712 (sshd brute force, level 10) fires on repeated invalid-user logins. From another host, try a nonexistent user against a monitored host several times:
```
for i in $(seq 1 10); do ssh -o BatchMode=yes -o ConnectTimeout=5 nosuchuser@<monitored-host> true; done
```
A real alert should travel the whole chain: agent, manager, rule match, integration script, ntfy, phone.

> **Gotcha:** I planned to use the Vikunja container as the target and wrote a to-do that its agent was missing, because it did not show in the dashboard yet. The agent was installed and had just not appeared, so that to-do was wrong. I got my alert by brute forcing the Caddy host instead.

## Step 7: Look at the alert volume
```
sudo jq -r 'select(.rule.level >= 10) | "\(.rule.level) \(.rule.id) \(.rule.description)"' /var/ossec/logs/alerts/alerts.json | sort | uniq -c | sort -rn | head -20
```
Mine showed web 400 errors (rule 31151), sshd brute force on nonexistent users (rule 5712) and common web attacks (rule 31153), all from my own testing. That is not a baseline.

## Open items
- There is no deduplication, so a burst of alerts means many pings. Rules 31151 and 5712 are already "multiple attempts" aggregates, but a campaign can still repeat.
- Rule 31151 may fire on public internet noise. If source IPs are logged as the public edge host's address, every hit would look like it comes from there. I have not checked.
- I need a day or two without testing, then a rerun of the count, to get a real baseline.
- Tune or silence noisy rules instead of raising the global threshold. A second integration block for levels 7 to 9, to a separate low-priority topic, is possible.

## Related Notes
- [ntfy](../services/ntfy.md)
- [Wazuh](../services/wazuh.md)
- [Deploy ntfy in a Debian LXC](deploy-ntfy-in-a-debian-lxc.md)
- [Run Wazuh alert triage](run-wazuh-alert-triage.md)

## Tags
#runbook
