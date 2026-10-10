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

This skill teaches the agent to wire alert routing end-to-end: notifiers (where alerts go), notification policies (how they're matched and routed), and muting rules (when to silence them). Setup, auth, `get`-before-`update`, safe delete, and common errors: see the oodle-cli skill.

## Quick Reference

| Resource | Commands |
|----------|----------|
| Notifiers | `oodle notifiers list \| get <id> \| create -f \| update <id> -f \| delete <id>` |
| Notification policies | `oodle notification-policies list \| get <id> \| create -f \| update <id> -f \| delete <id>` |
| Muting rules | `oodle muting-rules list \| get <id> \| create -f \| delete <id>` — no `update`: `get` it, `delete` it, and `create` the edited copy |

## Notifiers

A notifier is one destination. `type` is a number, and the config goes in the field for that type (there is no generic `config` field):

| `type` | Destination | Config field |
|--------|-------------|--------------|
| `0` | Email | `email_config` (`to`) |
| `1` | PagerDuty | `pagerduty_config` (`routing_key`) |
| `2` | Slack | `slack_config` (`api_url` = incoming webhook URL, `channel`) |
| `3` | Opsgenie | `opsgenie_config` |
| `4` | Webhook | `webhook_config` (`url`) |
| `5` | Google Chat | `googlechat_config` |
| `6` | Microsoft Teams | `msteamsv2_config` |
| `7` | Rootly | `rootly_config` |

```json
{
  "name": "slack-ops",
  "type": 2,
  "slack_config": { "api_url": "https://hooks.slack.com/services/T000/B000/XXXX", "channel": "#ops", "send_resolved": true }
}
```

Other types use the same shape, for example `{"name": "pd-platform", "type": 1, "pagerduty_config": {"routing_key": "<key>", "send_resolved": true}}`.

The create output has the notifier `id` (a UUID). Policies and monitors refer to notifiers by this id, not by name.

**Test a new notifier before you use it for critical alerts.** A wrong Slack webhook fails silently. Route a test monitor (low `warn` threshold) to the new notifier, confirm the message arrives, then add it to the production policy.

**Before you delete a notifier, find what uses it.** Alerts routed to a deleted notifier are dropped silently.

```bash
NID=<notifier-id>
oodle notification-policies list -o json | jq --arg n "$NID" '.[] | select([.notifiers[]?[]?] | index($n)) | {id, name}'
oodle monitors list -o json | jq --arg n "$NID" '.[] | select([.notifications[]?.notifiers[]?[]?] | index($n)) | {id, name}'
```

## Notification Policies and Monitor Routing

A policy lists the notifiers for each severity. Put the pager in `critical` and the chat channel in `warn` of one policy, instead of two policies that both match. `any` sends every severity. A policy with `"global": true` applies to all monitors, in addition to their own routing.

```json
{
  "name": "platform-prod",
  "notifiers": {
    "critical": ["<pagerduty-notifier-id>", "<slack-notifier-id>"],
    "warn": ["<slack-notifier-id>"],
    "no_data": ["<slack-notifier-id>"]
  }
}
```

A policy has no `matchers`, `receiver`, or `routes`. A monitor chooses where its alerts go in its `notifications` list. Each entry has label `matchers` and either a `notification_policy_id` or inline `notifiers`. The first entry whose matchers match the alert labels is used, so put specific entries first. An entry with empty matchers must be the last one.

```json
{
  "notifications": [
    {
      "matchers": [{"type": 0, "name": "team", "value": "platform"}, {"type": 0, "name": "env", "value": "prod"}],
      "notification_policy_id": "<policy-id>"
    },
    {
      "matchers": [],
      "notifiers": { "any": ["<slack-notifier-id>"] }
    }
  ]
}
```

Matcher `type` values (here and in muting rules): `0` = equals, `1` = not-equals, `2` = regex-match, `3` = regex-not-match.

To change the routing of a monitor, `get` it, set `.notifications` with `jq`, and `update` it.

## Muting Rules

A muting rule silences alerts whose labels match the matchers, between `startsAt` and `endsAt` (RFC 3339).

- **Always add the `_oodle_monitor_id` matcher.** Without it, the rule silences every alert that shares the other labels. Get the id from `oodle monitors list -o json | jq '.[] | {id,name}'`.
- **Always set `endsAt`, after `startsAt`.** An open-ended mute becomes a permanent silence that nobody remembers.

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

## Failure Handling

| Error | Cause | Fix |
|-------|-------|-----|
| Policy or monitor points to a missing notifier | The notifier id was deleted or is wrong | `oodle notifiers list -o json`; use the `id` (UUID), not the name |
| `only one of notification_policy_id ... or notifications should be set` | The monitor sets both the deprecated `notification_policy_id` and `notifications` | Keep only `notifications` |
| `notification without label matchers must be the last notification` | Catch-all entry is not last | Move the entry with empty matchers to the end |
| Slack messages not arriving | Wrong or deactivated webhook | Re-issue the webhook; update `slack_config.api_url`; test again |
| PagerDuty incidents not created | Wrong routing key or service | Confirm the key in PagerDuty; update `pagerduty_config.routing_key` |
| `unknown command "update"` for muting rules | Muting rules have no `update` | `get`, `delete`, and `create` the edited copy |
| Mute did not take effect | `startsAt` is in the future, or the matchers do not match the alert | `oodle muting-rules get <id> -o json`; compare the matchers with the alert labels |
