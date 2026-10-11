---
name: oodle-drop-rules
description: Create and manage Oodle metric drop rules — reduce ingestion cost by dropping high-volume, low-value metrics.
metadata:
  version: "1.1.0"
  author: oodle-ai
  repository: https://github.com/oodle-ai/agent-skills
  tags: oodle,drop-rules,cost,metrics
  globs: "**/*drop-rule*,**/*droprule*"
  alwaysApply: "false"
---

# Oodle Drop Rules — Cost Control

This skill teaches the agent to drop high-volume metrics safely: estimate the impact first, start with a narrow rule, and never silently delete a metric a dashboard depends on. There is no sampling mode: a rule drops every series that it matches, and dropped data cannot be recovered.

## Prerequisites

```bash
brew install oodle-ai/oodle/oodle
oodle configure
```

Confirm the drop-rules endpoint works:

```bash
oodle drop-rules list -o json | jq 'length'
```

## Command Execution Order

Before running any oodle command:
1. Check whether the target metric prefix and matchers are already in context.
2. If not, run `oodle metrics query --query 'count({__name__=~"<prefix>.*"})' -o json` to estimate impact.
3. Confirm no critical dashboards or monitors depend on the affected series (see "Estimating impact before creating a rule").
4. Start with a narrow rule (one metric, one non-production `env`) on the first attempt; widen the `filters` only after the dashboards and monitors of that env show no regression.
5. Run `oodle drop-rules create -f rule.json`.

## Quick Reference

| Task | Command |
|------|---------|
| List rules | `oodle drop-rules list -o json` |
| Get rule | `oodle drop-rules get <id> -o json` |
| Create rule | `oodle drop-rules create -f rule.json` |
| Update rule | `oodle drop-rules update <id> -f rule.json` |
| Delete rule | `oodle drop-rules delete <id> --force` |

## Common Operations

### Rule schema

```json
{
  "rule_name": "drop-noisy-debug-metrics",
  "metric_name": {"type": 2, "name": "__name__", "value": "debug_.*"},
  "filters": [
    {"type": 0, "name": "level", "value": "debug"},
    {"type": 0, "name": "env",   "value": "staging"}
  ],
  "type": "series"
}
```

| Field | Meaning |
|-------|---------|
| `rule_name` | Required. Name of the rule |
| `metric_name` | Matcher on the metric name (`"name": "__name__"`) |
| `filters` | Label matchers. All must match for the rule to fire (logical AND) |
| `type` | Required. Must be `"series"` |

Matcher `type`: `0` equals, `1` not equals, `2` regex match, `3` regex not match.

### Estimating impact before creating a rule

```bash
# ✅ CORRECT — count series that will be affected
oodle metrics query --query 'count({__name__=~"debug_.*"})' -o json

# ✅ CORRECT — confirm no dashboard panel queries the metric (list has no panels; get each dashboard)
for uid in $(oodle dashboards list -o json | jq -r '.[].uid'); do
  oodle dashboards get "$uid" -o json | grep -q 'debug_traffic_total' && echo "$uid"
done

# ✅ CORRECT — confirm no monitor depends on it
oodle monitors list -o json | jq '.[] | select(.promql_query | contains("debug_traffic_total"))'

# ❌ WRONG — create the rule first, find out from on-call later
oodle drop-rules create -f rule.json
```

### Creating a narrow rule (safer first step)

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

```bash
# ✅ CORRECT — one metric in staging; observe for a week before widening
oodle drop-rules create -f rule.json
```

### Widening a rule

```bash
# ✅ CORRECT — get → change filters → update
oodle drop-rules get dr_123 -o json > rule.json
jq '.filters = [{"type": 2, "name": "env", "value": "staging|dev"}]' rule.json > rule.new.json
oodle drop-rules update dr_123 -f rule.new.json

# ❌ WRONG — partial payload nulls metric_name and filters
oodle drop-rules update dr_123 -f <(echo '{"filters":[]}')
```

### Deleting (re-enabling ingestion)

```bash
# ✅ CORRECT
oodle drop-rules get dr_123 -o json > /dev/null
oodle drop-rules delete dr_123 --force

# ❌ WRONG — speculative delete by name match
oodle drop-rules delete "$(oodle drop-rules list | grep debug | awk '{print $1}')" --force
```

## Best Practices

### Estimate impact with `oodle metrics query --query 'count({__name__=~"<prefix>.*"})'` before creating a rule

A drop rule that matches more series than expected can hide real signal.

```bash
# ✅ CORRECT
oodle metrics query --query 'count({__name__=~"kube_pod_.*"})' -o json | jq -r '.data.result[0].value[1]'
# 1742 series — confirm with the team that all 1742 are safe to drop before creating a rule

# ❌ WRONG — create rule based on a guess; later discover a critical metric was matched
oodle drop-rules create -f rule.json
```

### Start with a narrow rule in a non-production env on the first iteration

There is no sampling mode. A narrow first rule shows that the metric truly is low-value before you drop it everywhere.

```bash
# ✅ CORRECT — week 1: drop in staging only, observe dashboards
"filters": [{"type": 0, "name": "env", "value": "staging"}]
# week 2: if no dashboards or monitors regressed, widen the filters

# ❌ WRONG — drop everywhere on first attempt; can break a dashboard nobody remembered
"filters": []
```

### Always include both `env` and `service` (or an exact `metric_name`) in matchers

Broad matchers like `{level: debug}` alone can match production metrics that happen to share a label.

```bash
# ✅ CORRECT — scoped to one env
"filters": [{"type":0,"name":"level","value":"debug"},{"type":0,"name":"env","value":"staging"}]

# ❌ WRONG — also drops debug metrics in prod
"filters": [{"type":0,"name":"level","value":"debug"}]
```

### Always `get` before `update` to preserve fields

Update is a full-document replace.

```bash
# ✅ CORRECT
oodle drop-rules get dr_123 -o json > rule.json
jq '.filters += [{"type": 0, "name": "service", "value": "api"}]' rule.json > rule.new.json
oodle drop-rules update dr_123 -f rule.new.json

# ❌ WRONG — drops metric_name and type
oodle drop-rules update dr_123 -f <(echo '{"filters":[{"type":0,"name":"service","value":"api"}]}')
```

### Name rules with `drop-<metric-or-domain>-<scope>`

Predictable names make it easy to find and revert a rule when a dashboard breaks.

```bash
# ✅ CORRECT
"rule_name": "drop-debug-metrics-staging"
"rule_name": "drop-kube-pod-info-prod"

# ❌ WRONG
"rule_name": "rule1"
```

## Failure Handling

| Error | Cause | Fix |
|-------|-------|-----|
| 401 Unauthorized | Invalid or missing API key | Run `oodle configure` or set `OODLE_API_KEY` |
| 404 Not Found | Drop rule ID does not exist | Verify with `oodle drop-rules list -o json` |
| connection refused | Wrong `OODLE_DEPLOYMENT` URL | Check `OODLE_DEPLOYMENT` env var |
| `rule_name cannot be empty` | Missing `rule_name` | Add `rule_name` |
| `type cannot be empty` / `invalid type "...", must be "series"` | `type` is missing or wrong | Set `"type": "series"` |
| Dashboard panel suddenly empty | Drop rule matched a metric the panel queries | Run `oodle drop-rules list -o json` to find the rule; delete it (`oodle drop-rules delete <id> --force`) or narrow its matchers |
| Monitor went into `no data` | Drop rule matched the monitor's metric | Same fix as above |
| Cost did not decrease | Matchers don't actually match the high-volume series | Re-run the `count(...)` query with the rule's matchers and compare label sets |
| 429 Too Many Requests | Bulk drop-rule sync | Add `--retries 3`, throttle to <10 creates per second |

## References

- [Oodle CLI repo](https://github.com/oodle-ai/oodle-cli)
- [Oodle docs](https://docs.oodle.ai)
