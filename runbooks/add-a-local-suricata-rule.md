---
tags:
  - runbook
  - "#suricata"
---

# Add a local Suricata rule

Adds a local rule that survives `suricata-update`, then triggers it to prove the whole path works: packets on the interface, a rule match, and an alert in the log. The rule alerts on the `curl` user agent, because the old `testmynids.org` test URL no longer resolves. See [Suricata](../services/suricata.md).

## Step 1: Write the rule
Local rule IDs start at 1000000, so they do not collide with the Emerging Threats set.
```
sudo mkdir -p /etc/suricata/rules
echo 'alert http any any -> any any (msg:"LOCAL test: curl user-agent"; http.user_agent; content:"curl/"; sid:1000001; rev:1;)' | sudo tee /etc/suricata/rules/local.rules
```

## Step 2: Tell suricata-update about the file
Without this, a plain run drops local rules, and only `--local` keeps them. `suricata-update --dump-sample-configs` writes sample files into the current directory instead of printing them, so run it in a scratch folder if you want to look. Do not copy the sample `update.yaml` into place, because it has `/etc/suricata/rules` active under `local:`, which loads every rules file in that directory.
```
sudo tee /etc/suricata/update.yaml <<'CONF'
local:
  - /etc/suricata/rules/local.rules
CONF
```

## Step 3: Update and check the merged file
The count should be `1`.
```
sudo suricata-update
grep -c 'sid:1000001' /var/lib/suricata/rules/suricata.rules
```

## Step 4: Test the config and restart
Restart so the running engine loads the merged file. Rules take about 30 seconds to load.
```
sudo suricata -T -c /etc/suricata/suricata.yaml -v
sudo systemctl restart suricata
sleep 30
```

## Step 5: Trigger the rule
Use plain HTTP. Suricata cannot read the contents of HTTPS traffic.
```
curl -s http://example.com -o /dev/null
sudo tail -n 3 /var/log/suricata/fast.log
```

## Verification
- `fast.log` has a line with `LOCAL test: curl user-agent`. If it is empty, wait ten seconds and check again.
- A plain `sudo suricata-update` still leaves the count at `1`, with no `--local` flag.
- `eve.json` in the same folder has the full JSON record for the alert.

## Related Notes
- [Suricata](../services/suricata.md)
- [Zeek](../services/zeek.md)

## Tags
#runbook
