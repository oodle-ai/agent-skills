---
name: oodle-alerting
description: Manage Oodle notifiers, notification policies, and muting rules — routing alerts to the right channels and silencing during maintenance.
metadata:
  version: "1.1.0"
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
| Change muting rule | No `update`: `get` it, `delete` it, and `create` the edited copy |
| Delete muting rule | `oodle muting-rules delete <id> --force` |

## Common Operations

### Notifiers — channel definitions

A notifier wraps a destination (Slack, PagerDuty, email, webhook). `type` is a number, and the config goes in the field for that type:

| `type` | Destination | Config field |
|--------|-------------|--------------|
| `0` | Email | `email_config` |
| `1` | PagerDuty | `pagerduty_config` |
| `2` | Slack | `slack_config` |
| `3` | Opsgenie | `opsgenie_config` |
| `4` | Webhook | `webhook_config` |
| `5` | Google Chat | `googlechat_config` |
| `6` | Microsoft Teams | `msteamsv2_config` |
| `7` | Rootly | `rootly_config` |

Slack:

```json
{
  "name": "slack-ops",
  "type": 2,
  "slack_config": { "api_url": "https://hooks.slack.com/services/T000/B000/XXXX", "channel": "#ops", "send_resolved": true }
}
```

PagerDuty:

```json
{
  "name": "pd-platform",
  "type": 1,
  "pagerduty_config": { "routing_key": "abc123def456", "send_resolved": true }
}
```

Email:

```json
{
  "name": "email-platform",
  "type": 0,
  "email_config": { "to": "oncall@example.com", "send_resolved": true }
}
```

Webhook:

```json
{
  "name": "webhook-incident-bot",
  "type": 4,
  "webhook_config": { "url": "https://incidents.example.com/oodle", "send_resolved": true }
}
```

The create output has the notifier `id` (a UUID). Policies and monitors refer to notifiers by this id, not by name.

```bash
# ✅ CORRECT
oodle notifiers create -f notifier.json

# ❌ WRONG — missing config field, server returns 400
oodle notifiers create -f <(echo '{"name":"x","type":2}')
```

### Notification policies — routing

A notification policy lists the notifiers (by id) for each severity: `critical`, `warn`, `no_data`, or `any`. It has no label matchers.

```json
{
  "name": "platform-prod",
  "notifiers": {
    "critical": ["<pd-platform-notifier-id>", "<slack-ops-notifier-id>"],
    "warn": ["<slack-ops-notifier-id>"]
  }
}
```

A monitor matches its alerts by labels and routes them in its `notifications` list. Each entry has label `matchers` and a `notification_policy_id` (or inline `notifiers`). The first entry whose matchers match is used. An entry with empty matchers must be the last one.

```json
"notifications": [
  {
    "matchers": [{"type": 0, "name": "team", "value": "platform"}, {"type": 0, "name": "env", "value": "prod"}],
    "notification_policy_id": "<policy-id>"
  }
]
```

```bash
# ✅ CORRECT
oodle notification-policies create -f policy.json

# ❌ WRONG — references a notifier id that doesn't exist
# ("critical": ["non-existent"])
```

### Muting rules — scheduled silence

Muting rules silence alerts whose labels match the matchers, between `startsAt` and `endsAt` (RFC 3339).

**Always scope a muting rule to a specific monitor** by including the `_oodle_monitor_id` matcher. Without it, the rule can silence unrelated alerts that happen to share the other labels. Resolve the monitor ID first with `oodle monitors list -o json` (or `oodle monitors get <id>`).

Matcher `type` values: `0` = equals, `1` = not-equals, `2` = regex-match, `3` = regex-not-match.

```json
{
  "name": "deploy-window-2024-01-15",
  "matchers": [
    {"type": 0, "name": "_oodle_monitor_id", "value": "01998476-6dd3-73e8-a4c0-d9137eef6bc9"},
    {"type": 0, "name": "service", "value": "api"},
    {"type": 0, "name": "env",     "value": "prod"}
  ],
  "startsAt": "2024-01-15T02:00:00Z",
  "endsAt":   "2024-01-15T06:00:00Z"
}
```

```bash
# ✅ CORRECT — bounded window, scoped to a specific monitor
oodle muting-rules create -f mute.json

# ❌ WRONG — endsAt before startsAt (rejected by server)
# {"startsAt": "2024-01-15T06:00:00Z", "endsAt": "2024-01-15T02:00:00Z"}

# ❌ WRONG — no _oodle_monitor_id matcher; silences every alert with these labels
# "matchers": [{"type":0,"name":"service","value":"api"},{"type":0,"name":"env","value":"prod"}]
```

## Best Practices

### Test a new notifier with a low-severity monitor before wiring it to a critical alert

A misconfigured Slack webhook fails silently — alerts disappear into the void.

```bash
# ✅ CORRECT — create notifier, attach to a test monitor with a low `warn` threshold, fire it once,
# confirm message arrives, then attach to the production policy.
oodle notifiers create -f notifier.json
oodle monitors create -f test-monitor.json   # low `warn` threshold, routed to the new notifier
# wait for fire, confirm receipt, then:
oodle notification-policies update pol_123 -f policy.json   # adds the new notifier

# ❌ WRONG — attach an untested notifier directly to the `critical` severity of a policy
oodle notifiers create -f notifier.json
oodle notification-policies create -f policy-critical.json
```

### Always `get` a notifier before deleting it — check which policies reference it

Deleting a notifier referenced by an active policy causes alerts to be dropped silently.

```bash
# ✅ CORRECT — find references first
oodle notifiers get notif_slack_ops -o json
oodle notification-policies list -o json | jq '.[] | select([.notifiers[]?[]?] | index("notif_slack_ops")) | .id'
oodle monitors list -o json | jq '.[] | select([.notifications[]?.notifiers[]?[]?] | index("notif_slack_ops")) | .id'
# update those policies and monitors to a different notifier, then:
oodle notifiers delete notif_slack_ops --force

# ❌ WRONG — delete without checking; live policies suddenly route to a missing notifier
oodle notifiers delete notif_slack_ops --force
```

### Always include `_oodle_monitor_id` in muting rule matchers

A mute without `_oodle_monitor_id` silences every alert whose labels happen to match — frequently far more than intended. Pin it to the monitor being silenced.

```bash
# ✅ CORRECT — scoped to one monitor
"matchers": [
  {"type":0,"name":"_oodle_monitor_id","value":"01998476-6dd3-73e8-a4c0-d9137eef6bc9"},
  {"type":0,"name":"service","value":"api"}
]

# ❌ WRONG — any monitor with service=api gets silenced
"matchers": [{"type":0,"name":"service","value":"api"}]
```

Resolve the monitor ID with `oodle monitors list -o json | jq '.[] | {id,name}'` before authoring the rule.

### Always set `endsAt` on muting rules — never create open-ended mutes

An open-ended mute that nobody remembers becomes a permanent silence on a critical alert.

```bash
# ✅ CORRECT — bounded mute window
"startsAt": "2024-01-15T02:00:00Z", "endsAt": "2024-01-15T06:00:00Z"

# ❌ WRONG — open-ended; alerts stay silenced forever
"startsAt": "2024-01-15T02:00:00Z", "endsAt": null
```

### Scope monitor `notifications` matchers to the smallest set of labels that uniquely identify the team

Overly-broad matchers (`{env: prod}` only) route every prod alert to one notifier.

```bash
# ✅ CORRECT
"matchers": [{"type":0,"name":"team","value":"platform"},{"type":0,"name":"env","value":"prod"}]

# ❌ WRONG — every prod alert routes here
"matchers": [{"type":0,"name":"env","value":"prod"}]
```

### Use the severities of one policy for escalation, not separate policies

Put the pager in `critical` and the chat channel in `warn` of one policy. Two policies that both apply to an alert will double-fire.

```bash
# ✅ CORRECT
"notifiers": {"critical": ["<pd-platform-notifier-id>"], "warn": ["<slack-ops-notifier-id>"]}

# ❌ WRONG — two policies, both apply, both fire
```

## Failure Handling

| Error | Cause | Fix |
|-------|-------|-----|
| 401 Unauthorized | Invalid or missing API key | Run `oodle configure` or set `OODLE_API_KEY` |
| 404 Not Found | Notifier / policy / muting-rule ID does not exist | Verify with the matching `list` command |
| connection refused | Wrong `OODLE_DEPLOYMENT` URL | Check `OODLE_DEPLOYMENT` env var |
| Policy or monitor points to a missing notifier | The notifier id was deleted or is wrong | Run `oodle notifiers list -o json` and use the `id` (UUID), not the name |
| `notification without label matchers must be the last notification` | Catch-all entry in a monitor's `notifications` is not last | Move the entry with empty matchers to the end |
| Slack messages not arriving | Wrong webhook URL or webhook deactivated | Re-issue the Slack incoming webhook; update notifier `slack_config.api_url`; re-test |
| PagerDuty incidents not created | Wrong `routing_key` or wrong service | Confirm the routing key in PagerDuty; update notifier `pagerduty_config.routing_key` |
| `oodle muting-rules update` prints the help | Muting rules have no `update` | `get`, `delete`, and `create` the edited copy |
| Mute didn't take effect | `startsAt` is in the future or label matchers don't match the alert | `oodle muting-rules get <id> -o json` and compare matchers to the firing alert's labels |
| 429 Too Many Requests | Bulk policy sync | Add `--retries 3`, throttle to <10 creates per second |

## References

- [Oodle CLI repo](https://github.com/oodle-ai/oodle-cli)
- [Oodle docs](https://docs.oodle.ai)
