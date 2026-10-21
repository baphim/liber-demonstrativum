---
tags:
  - runbook
  - "#vikunja"
  - "#ntfy"
  - "#systemd"
---

# Send a Vikunja digest to ntfy

A systemd timer on the Vikunja host runs a script every morning. It pushes the tasks due today and a one line overdue summary to my phone through [ntfy](../services/ntfy.md), and sends nothing on empty days. Values in angle brackets are placeholders. Needs `jq`.

## Why a polling script
Vikunja has no native mobile push. I looked at email, the PWA, CalDAV with a tasks app, user webhooks to ntfy, and a polling script. Webhooks do not retry failed deliveries, they send JSON that needs a relay to become a readable message, and Vikunja blocks outgoing requests to non-routable IPs by default. A polling script run by a timer avoids all three.

## Step 1: Publish-only ntfy user
Create a bot user with write-only access to one topic, as described in [Deploy ntfy in a Debian LXC](deploy-ntfy-in-a-debian-lxc.md). The script uses its token.

## Step 2: Read-only Vikunja API token
In Vikunja, Settings, API Tokens, create a read-only token. Check the endpoint on your version:
```
curl -H "Authorization: Bearer <token>" "https://vikunja.<tailnet>.ts.net/api/v1/tasks?per_page=1"
```
It should return 200.

The task list takes `filter`, `filter_timezone`, `sort_by`, `order_by` and `per_page`. Date math uses `now`, `now/d` (start of today) and `now+7d`. With `filter_timezone` set, "today" starts at local midnight. The saved filters in the web UI cannot pass that parameter, so they use the server timezone.

> **Gotcha:** I pasted a multi-line `curl` once and got `{"code":11,"message":"missing, malformed, expired or otherwise invalid token provided"}`. The same call as a single line worked. I never confirmed what broke.

> **Gotcha:** A task due at exactly 00:00 did not match `due_date >= now/d`. The same task moved to 23:59 did. Imported tasks with date-only due dates sit at midnight, so the script uses a lower bound of `now/d-30s`. Midnight tasks then match whichever way the boundary behaves, and overdue uses the same cutoff so no task shows up in both lists.

## Step 3: Secrets file
Put the two tokens in a root-only file as plain `KEY=value` lines, for systemd's `EnvironmentFile`:
```
install -m 600 -o root -g root /dev/null /etc/vikunja-digest.env
nano /etc/vikunja-digest.env
```
```
VIKUNJA_TOKEN=<vikunja-token>
NTFY_TOKEN=<ntfy-bot-token>
```
> **Gotcha:** Do not `source` this file in a shell. Sourcing evaluates it as shell code. One of my lines had no `=` sign, so bash tried to run it as a command and printed the token in the "command not found" error. I never confirmed how the line ended up that way. I did not rotate that token, because it is read-only and the instance is only reachable over the tailnet. I cleared the scrollback afterward.

This checks the structure without printing any values:
```
awk '{ ok = ($0 ~ /^[A-Z_]+=[^[:space:]]+$/) ? "ok" : "BAD"; name = index($0,"=") ? substr($0,1,index($0,"=")-1) : "(no = sign)"; print NR, ok, name, length($0) }' /etc/vikunja-digest.env
```
This loads the file into the shell for testing without evaluating anything:
```
while IFS='=' read -r k v; do [ -n "$k" ] && export "$k=$v"; done < /etc/vikunja-digest.env
echo "${#VIKUNJA_TOKEN} ${#NTFY_TOKEN}"
```

## Step 4: The script
`/usr/local/bin/vikunja-digest.sh`, mode 700. `--dry-run` prints the digest instead of sending it. It shows up to 5 tasks due today, then "+N more due today", and up to 3 overdue titles, then "+N more". If the Vikunja call fails, it sends a high priority "Vikunja digest failed" notification, so an expired token does not fail silently.
```bash
#!/usr/bin/env bash
# Daily Vikunja digest to ntfy. Use --dry-run to print instead of sending.
set -uo pipefail

VIKUNJA_URL="https://vikunja.<tailnet>.ts.net"
TASKS_PATH="/api/v1/tasks"
NTFY_URL="https://ntfy.<tailnet>.ts.net/<topic>"
TZ_NAME="<Region/City>"
MAX_TODAY=5
MAX_OVERDUE=3
DRY_RUN=0
[ "${1:-}" = "--dry-run" ] && DRY_RUN=1

: "${VIKUNJA_TOKEN:?VIKUNJA_TOKEN not set}"
[ "$DRY_RUN" -eq 1 ] || : "${NTFY_TOKEN:?NTFY_TOKEN not set}"

notify() { # title, message, priority, tags
  if [ "$DRY_RUN" -eq 1 ]; then
    printf 'TITLE: %s\nPRIORITY: %s\n---\n%s\n' "$1" "${3:-default}" "$2"
    return 0
  fi
  curl -fsS -H "Authorization: Bearer $NTFY_TOKEN" \
    -H "Title: $1" -H "Priority: ${3:-default}" -H "Tags: ${4:-calendar}" \
    -d "$2" "$NTFY_URL" >/dev/null
}

fail() {
  notify "Vikunja digest failed" "$1" high warning
  exit 1
}

fetch() { # filter query -> JSON array on stdout
  local out
  out=$(curl -fsS -G "$VIKUNJA_URL$TASKS_PATH" \
    -H "Authorization: Bearer $VIKUNJA_TOKEN" \
    --data-urlencode "filter=$1" \
    --data-urlencode "filter_timezone=$TZ_NAME" \
    --data-urlencode "sort_by=due_date" \
    --data-urlencode "order_by=asc" \
    --data-urlencode "per_page=50") || return 1
  jq -e 'type == "array"' >/dev/null <<<"$out" || return 1
  printf '%s' "$out"
}

today=$(fetch 'done = false && due_date >= now/d-30s && due_date < now/d+1d') \
  || fail "Could not fetch today's tasks from Vikunja."
overdue=$(fetch 'done = false && due_date < now/d-30s') \
  || fail "Could not fetch overdue tasks from Vikunja."

n_today=$(jq 'length' <<<"$today")
n_over=$(jq 'length' <<<"$overdue")

if [ "$n_today" -eq 0 ] && [ "$n_over" -eq 0 ]; then
  [ "$DRY_RUN" -eq 1 ] && echo "Nothing due or overdue; no notification would be sent."
  exit 0
fi

body=""
if [ "$n_today" -gt 0 ]; then
  body=$(jq -r --argjson max "$MAX_TODAY" '.[:$max][] | "- " + .title' <<<"$today")
  if [ "$n_today" -gt "$MAX_TODAY" ]; then
    body+=$'\n'"+$((n_today - MAX_TODAY)) more due today"
  fi
fi
if [ "$n_over" -gt 0 ]; then
  titles=$(jq -r --argjson max "$MAX_OVERDUE" '[.[:$max][] | .title] | join(", ")' <<<"$overdue")
  line="Overdue ($n_over): $titles"
  if [ "$n_over" -gt "$MAX_OVERDUE" ]; then
    line+=" +$((n_over - MAX_OVERDUE)) more"
  fi
  [ -n "$body" ] && body+=$'\n\n'
  body+="$line"
fi

if [ "$n_today" -gt 0 ]; then title="Due today: $n_today"; else title="Overdue: $n_over"; fi
notify "$title" "$body" default calendar || exit 1
```
Run it with `--dry-run` first, then once for real. The real run should exit 0 and the notification should reach the phone.

## Step 5: systemd service and timer
`/etc/systemd/system/vikunja-digest.service`:
```ini
[Unit]
Description=Vikunja daily digest to ntfy
After=network-online.target tailscaled.service
Wants=network-online.target

[Service]
Type=oneshot
EnvironmentFile=/etc/vikunja-digest.env
ExecStart=/usr/local/bin/vikunja-digest.sh
```
`/etc/systemd/system/vikunja-digest.timer`:
```ini
[Unit]
Description=Vikunja digest at 9:30 AM local time

[Timer]
OnCalendar=*-*-* 09:30:00 <Region/City>
Persistent=true

[Install]
WantedBy=timers.target
```
```
systemd-analyze calendar "*-*-* 09:30:00 <Region/City>"
systemctl daemon-reload
systemctl enable --now vikunja-digest.timer
systemctl list-timers vikunja-digest.timer
systemctl start vikunja-digest.service
journalctl -u vikunja-digest.service -n 20 --no-pager
```
The time zone is part of the `OnCalendar` expression, so the schedule follows local time across daylight saving changes. `Persistent=true` runs it on the next start if the container was off at the scheduled time. The manual `start` tests the real service, which a shell test cannot, because systemd loads the env file itself.

## Not verified
- I did not record whether the first scheduled run arrived.
- The saved Overdue filter in the Vikunja web UI uses the server timezone. If `service.timezone` is unset, "today" there can start at the wrong hour. I have not checked it.

## Related Notes
- [ntfy](../services/ntfy.md)
- [Vikunja](../services/vikunja.md)
- [Deploy ntfy in a Debian LXC](deploy-ntfy-in-a-debian-lxc.md)

## Tags
#runbook
