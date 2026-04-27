---
name: oodle-alerting
description: Manage Oodle notifiers, notification policies, and muting rules — routing alerts to the right channels and silencing during maintenance.
metadata:
  version: "1.0.0"
  author: oodle-ai
  repository: https://github.com/oodle-ai/agent-skills
  tags: oodle,alerting,notifiers,notifications,muting
  globs: "**/*notif*,**/*alert*,**/*mute*"
  alwaysApply: "false"
---

# Oodle Alerting — Notifiers, Policies, Muting

This skill teaches the agent to wire alert routing end-to-end: notifiers (where alerts go), notification policies (how they're matched and routed), and muting rules (when to silence them).

## Prerequisites

```bash
brew install oodle-ai/oodle/oodle
oodle configure
```

Verify all three subsystems respond:

```bash
oodle notifiers list -o json | jq 'length'
oodle notification-policies list -o json | jq 'length'
oodle muting-rules list -o json | jq 'length'
```

## Command Execution Order

Before running any oodle command:
1. Check whether the required resource ID or name is already in context.
2. If not, run the discovery command (e.g., `oodle notifiers list -o json`).
3. If the result is ambiguous, ask the user to confirm before proceeding.
4. Run the target command with the resolved ID.
5. Do not run speculative commands (e.g., do not `delete` a notifier without checking which policies reference it).

## Quick Reference

| Task | Command |
|------|---------|
| List notifiers | `oodle notifiers list -o json` |
| Get notifier | `oodle notifiers get <id> -o json` |
| Create notifier | `oodle notifiers create -f notifier.json` |
| Update notifier | `oodle notifiers update <id> -f notifier.json` |
| Delete notifier | `oodle notifiers delete <id> --force` |
| List policies | `oodle notification-policies list -o json` |
| Get policy | `oodle notification-policies get <id> -o json` |
| Create policy | `oodle notification-policies create -f policy.json` |
| Update policy | `oodle notification-policies update <id> -f policy.json` |
| Delete policy | `oodle notification-policies delete <id> --force` |
| List muting rules | `oodle muting-rules list -o json` |
| Get muting rule | `oodle muting-rules get <id> -o json` |
| Create muting rule | `oodle muting-rules create -f mute.json` |
| Update muting rule | `oodle muting-rules update <id> -f mute.json` |
| Delete muting rule | `oodle muting-rules delete <id> --force` |

## Common Operations

### Notifiers — channel definitions

A notifier wraps a destination (Slack, PagerDuty, email, webhook). The `config` block depends on `type`.

Slack:

```json
{
  "name": "slack-ops",
  "type": "slack",
  "config": { "webhookUrl": "https://hooks.slack.com/services/T000/B000/XXXX" }
}
```

PagerDuty:

```json
{
  "name": "pd-platform",
  "type": "pagerduty",
  "config": { "integrationKey": "abc123def456" }
}
```

Email:

```json
{
  "name": "email-platform",
  "type": "email",
  "config": { "addresses": ["oncall@example.com", "alerts@example.com"] }
}
```

Webhook:

```json
{
  "name": "webhook-incident-bot",
  "type": "webhook",
  "config": { "url": "https://incidents.example.com/oodle" }
}
```

```bash
# ✅ CORRECT
oodle notifiers create -f notifier.json

# ❌ WRONG — missing config block, server returns 400
oodle notifiers create -f <(echo '{"name":"x","type":"slack"}')
```

### Notification policies — routing

A notification policy matches alerts by labels and routes them to a receiver (notifier name).

```json
{
  "name": "platform-prod",
  "matchers": [
    {"name": "team", "value": "platform"},
    {"name": "env",  "value": "prod"}
  ],
  "receiver": "slack-ops",
  "routes": [
    {
      "matchers": [{"name": "severity", "value": "critical"}],
      "receiver": "pd-platform"
    }
  ]
}
```

```bash
# ✅ CORRECT
oodle notification-policies create -f policy.json

# ❌ WRONG — references a notifier that doesn't exist; server returns 400
# (no `receiver` set or `receiver: "non-existent"`)
```

### Muting rules — scheduled silence

Muting rules silence alerts whose labels match the matchers, between `startsAt` and `endsAt` (RFC 3339).

```json
{
  "name": "deploy-window-2024-01-15",
  "matchers": [
    {"name": "service", "value": "api"},
    {"name": "env",     "value": "prod"}
  ],
  "startsAt": "2024-01-15T02:00:00Z",
  "endsAt":   "2024-01-15T06:00:00Z"
}
```

```bash
# ✅ CORRECT — bounded window
oodle muting-rules create -f mute.json

# ❌ WRONG — endsAt before startsAt (rejected by server)
# {"startsAt": "2024-01-15T06:00:00Z", "endsAt": "2024-01-15T02:00:00Z"}
```

## Best Practices

### Test a new notifier with a low-severity monitor before wiring it to a critical alert

A misconfigured Slack webhook fails silently — alerts disappear into the void.

```bash
# ✅ CORRECT — create notifier, attach to a `severity=info` test monitor, fire it once,
# confirm message arrives, then attach to the production policy.
oodle notifiers create -f notifier.json
oodle monitors create -f test-monitor.json   # severity=info, low threshold
# wait for fire, confirm receipt, then:
oodle notification-policies update pol_123 -f policy.json   # adds the new notifier

# ❌ WRONG — attach an untested notifier directly to a `severity=critical` policy
oodle notifiers create -f notifier.json
oodle notification-policies create -f policy-critical.json
```

### Always `get` a notifier before deleting it — check which policies reference it

Deleting a notifier referenced by an active policy causes alerts to be dropped silently.

```bash
# ✅ CORRECT — find references first
oodle notifiers get notif_slack_ops -o json
oodle notification-policies list -o json | jq '.[] | select(.receiver=="slack-ops" or (.routes[]?.receiver=="slack-ops")) | .id'
# update those policies to a different receiver, then:
oodle notifiers delete notif_slack_ops --force

# ❌ WRONG — delete without checking; live policies suddenly route to a missing receiver
oodle notifiers delete notif_slack_ops --force
```

### Always set `endsAt` on muting rules — never create open-ended mutes

An open-ended mute that nobody remembers becomes a permanent silence on a critical alert.

```bash
# ✅ CORRECT — bounded mute window
"startsAt": "2024-01-15T02:00:00Z", "endsAt": "2024-01-15T06:00:00Z"

# ❌ WRONG — open-ended; alerts stay silenced forever
"startsAt": "2024-01-15T02:00:00Z", "endsAt": null
```

### Scope policy matchers to the smallest set of labels that uniquely identify the team

Overly-broad matchers (`{env: prod}` only) route every prod alert to one notifier.

```bash
# ✅ CORRECT
"matchers": [{"name":"team","value":"platform"},{"name":"env","value":"prod"}]

# ❌ WRONG — every prod alert in the org routes here
"matchers": [{"name":"env","value":"prod"}]
```

### Use nested `routes` for severity escalation, not separate top-level policies

A child route inherits the parent matchers; defining a separate top-level policy for `severity=critical` will double-fire.

```bash
# ✅ CORRECT
"matchers": [{"name":"team","value":"platform"}],
"receiver": "slack-ops",
"routes": [{"matchers":[{"name":"severity","value":"critical"}],"receiver":"pd-platform"}]

# ❌ WRONG — two top-level policies, both match, both fire
```

## Failure Handling

| Error | Cause | Fix |
|-------|-------|-----|
| 401 Unauthorized | Invalid or missing API key | Run `oodle configure` or set `OODLE_API_KEY` |
| 404 Not Found | Notifier / policy / muting-rule ID does not exist | Verify with the matching `list` command |
| connection refused | Wrong `OODLE_DEPLOYMENT` URL | Check `OODLE_DEPLOYMENT` env var |
| `receiver not found` | Policy references a notifier name that doesn't exist | Run `oodle notifiers list -o json` and match the `name` field exactly |
| Slack messages not arriving | Wrong webhook URL or webhook deactivated | Re-issue the Slack incoming webhook; update notifier `config.webhookUrl`; re-test |
| PagerDuty incidents not created | Wrong `integrationKey` or wrong service | Confirm the routing key in PagerDuty; update notifier `config.integrationKey` |
| Mute didn't take effect | `startsAt` is in the future or label matchers don't match the alert | `oodle muting-rules get <id> -o json` and compare matchers to the firing alert's labels |
| 429 Too Many Requests | Bulk policy sync | Add `--retries 3`, throttle to <10 creates per second |

## References

- [Oodle CLI repo](https://github.com/oodle-ai/oodle-cli)
- [Oodle docs](https://docs.oodle.ai)
