---
name: oodle-drop-rules
description: Create and manage Oodle metric drop rules — reduce ingestion cost by dropping the series of high-volume, low-value metrics at ingest.
metadata:
  version: "1.1.0"
  author: oodle-ai
  repository: https://github.com/oodle-ai/agent-skills
  tags: oodle,drop-rules,cost,metrics
  globs: "**/*drop-rule*,**/*droprule*"
  alwaysApply: "false"
---

# Oodle Drop Rules — Cost Control

This skill teaches the agent to drop high-volume metric series safely: estimate the impact first, scope the rule tightly, and never silently delete a metric a dashboard or monitor depends on.

A drop rule discards matching series when they arrive. Dropped data is not stored and cannot be recovered. There is no sampling mode: a rule drops every series that it matches. Deleting a rule restores ingestion from that time on; the data dropped before stays lost.

## Prerequisites

```bash
brew install oodle-ai/oodle/oodle
oodle configure   # or set OODLE_API_KEY, OODLE_INSTANCE, and OODLE_DEPLOYMENT

# Check access
oodle drop-rules list -o json | jq 'length'
```

Other install methods, auth details, and common errors: see the oodle-cli skill.

## Workflow

1. Count the series that the rule would match.
2. Confirm that no dashboard or monitor uses them. If one does, ask the owner before you continue.
3. Start small. There is no sampling mode, so the safe first step is a narrow rule: one metric in a non-production env (`env="staging"`). Watch the dashboards and monitors of that env, then widen the `filters`.
4. `oodle drop-rules create -f rule.json`.
5. Confirm the drop: after a few minutes, the `count(...)` query below returns no result.

`oodle drop-rules` has `list`, `get <id>`, `create -f`, `update <id> -f`, and `delete <id>`.

## Rule Schema

```json
{
  "rule_name": "drop-debug-traffic-staging",
  "metric_name": {"type": 0, "name": "__name__", "value": "debug_traffic_total"},
  "filters": [
    {"type": 0, "name": "env", "value": "staging"}
  ],
  "type": "series"
}
```

| Field | Meaning |
|-------|---------|
| `rule_name` | Required. Use `drop-<metric-or-domain>-<scope>`, so the rule is easy to find and revert when a dashboard breaks. |
| `metric_name` | Matcher on the metric name. Prefer an exact name: a regex such as `debug_.*` can match metrics added later. |
| `filters` | Label matchers, all must match (AND). Always scope with `env`, and with `service` when the metric is shared. An empty list drops the metric everywhere, including production. |
| `type` | Required. Must be `"series"`. |

Matcher `type` values: `0` = equals, `1` = not-equals, `2` = regex-match, `3` = regex-not-match. A regex must match the whole value. `name`, `matchers`, `action`, and `sampleRate` are not fields.

## Estimating Impact

```bash
# Series that the rule will drop
oodle metrics query --query 'count({__name__="debug_traffic_total", env="staging"})' -o json

# Monitors that query the metric
oodle monitors list -o json | jq '.[] | select(.promql_query | contains("debug_traffic_total")) | {id, name}'

# Dashboards that query the metric (list has no panels; get each one)
for uid in $(oodle dashboards list -o json | jq -r '.[].uid'); do
  oodle dashboards get "$uid" -o json | grep -q 'debug_traffic_total' && echo "$uid"
done
```

## Changing or Deleting a Rule

`update` replaces the whole rule. A body with only the changed field fails or removes the other fields. Always `get` → edit → `update`:

```bash
# ✅ CORRECT
oodle drop-rules get <id> -o json > rule.json
jq '.filters += [{"type": 0, "name": "service", "value": "api"}]' rule.json > rule.new.json
oodle drop-rules update <id> -f rule.new.json

# ❌ WRONG — partial body
oodle drop-rules update <id> -f <(echo '{"filters":[]}')
```

To restore ingestion, delete the rule after you confirm its ID: `oodle drop-rules get <id> -o json`, then `oodle drop-rules delete <id> --force`. Data dropped before the delete stays lost.

## Failure Handling

| Error | Cause | Fix |
|-------|-------|-----|
| `rule_name cannot be empty` | Missing `rule_name` | Add `rule_name` |
| `type cannot be empty` / `invalid type "...", must be "series"` | `type` is missing or wrong | Set `"type": "series"` |
| Dashboard panel suddenly empty, or monitor in `no data` | A drop rule matched its metric | Find the rule with `oodle drop-rules list -o json`; delete it or narrow its `filters` |
| Cost did not decrease | The matchers do not match the high-volume series | Run the `count(...)` query with the same matchers; compare label sets |
