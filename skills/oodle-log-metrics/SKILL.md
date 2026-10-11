---
name: oodle-log-metrics
description: Create and manage Oodle log-based metric rules — extract metrics from log streams using filters and labels. This is how to count, or alert on, values in log lines.
metadata:
  version: "1.1.0"
  author: oodle-ai
  repository: https://github.com/oodle-ai/agent-skills
  tags: oodle,log-metrics,logs,metrics
  globs: "**/*log-metric*,**/*logmetric*"
  alwaysApply: "false"
---

# Oodle Log Metrics — Rules and Cardinality

This skill teaches the agent to convert log streams into metrics safely: validate the filter, narrow the `labels`, and avoid creating high-cardinality time series. A log metrics rule is also the way to count or alert on values inside log lines: a monitor alerts on the metric that the rule emits.

## Prerequisites

```bash
brew install oodle-ai/oodle/oodle
oodle configure
```

Confirm the log-metrics endpoint works:

```bash
oodle log-metrics list -o json | jq 'length'
```

## Command Execution Order

Before running any oodle command:
1. Check whether the rule's filter and `labels` are already in context.
2. If not, run a sample log query (e.g. `oodle logs query`, see the oodle-logs skill, or the log explorer) to verify the filter actually matches logs.
3. Estimate cardinality: the distinct values of each label, multiplied together.
4. Run `oodle log-metrics create -f rule.json` only after the filter is validated.
5. Do not run `update` without first running `get` to capture the existing rule.

## Quick Reference

| Task | Command |
|------|---------|
| List rules | `oodle log-metrics list -o json` |
| Get rule | `oodle log-metrics get <id> -o json` |
| Create rule | `oodle log-metrics create -f rule.json` |
| Update rule | `oodle log-metrics update <id> -f rule.json` |
| Delete rule | `oodle log-metrics delete <id> --force` |

## Common Operations

### Rule schema

```json
{
  "name": "http_errors_from_logs",
  "filter": {
    "all": [
      {"field": "level", "operator": "is", "value": "error"},
      {"field": "service", "operator": "is", "value": "api"}
    ]
  },
  "labels": [
    {"name": "service", "valueExtractor": {"field": "service"}},
    {"name": "env", "valueExtractor": {"field": "env"}}
  ],
  "metricDefinitions": [
    {"name": "http_errors", "type": "log_count"}
  ]
}
```

Field meaning:

| Field | Meaning |
|-------|---------|
| `name` | Human identifier for the rule. Required. |
| `filter` | Which log lines the rule reads. A condition `{"field", "operator", "value", "jsonPath"}`, or `{"all": [...]}` (AND), `{"any": [...]}` (OR), `{"not": <filter>}`. You can nest them. Operators: `is`, `contains`, `matches regex`, `exists`. There is no `!=`; use `not`. |
| `labels` | Labels added to the emitted metrics. Each has a `name` and a static `value`, or a `valueExtractor` `{"field", "jsonPath", "regex"}`. |
| `metricDefinitions` | The metrics to emit. At least one. Each has a `name` and a `type`: `log_count` (number of matching lines), `counter` (sum of a value), `gauge` (last value), or `histogram` (distribution of a value). `counter`, `gauge`, and `histogram` read the value from `field`, with an optional `jsonPath` or `regex`. |

- `field` is a top-level log field: letters, digits, and underscores only. Read a nested JSON value with `jsonPath`, for example `{"field": "log", "jsonPath": "user.id"}`.
- A `regex` uses Rust syntax. Capture group 1 is the value, so put one group around the part you need. In JSON, escape each backslash: `\d` is `"\\d"`.
- When the regex does not match a line, the label is empty, and a value is `0`. So make the filter select only the lines that contain the value.
- The emitted metric name is `oodle_logs_<name>` (`oodle_logs_http_errors`). A `histogram` emits `_bucket`, `_sum`, and `_count` series. Query `log_count` and `counter` with `increase(...)` or `rate(...)`, and aggregate with `sum by (...)`.
- A rule applies only to logs that arrive after you create it. For past data, use `oodle logs query`.
- Put a single condition inside an `all` list, as in the examples. That form works with every CLI and server version.

### Creating a rule

```bash
# ✅ CORRECT — validate filter first, then create
# (preview with the log explorer or `oodle logs query`)
oodle log-metrics create -f rule.json

# ❌ WRONG — creating with an untested filter; the resulting metric is silently empty
oodle log-metrics create -f <(echo '{"name":"x","filter":{"all":[{"field":"levl","operator":"is","value":"eror"}]},"metricDefinitions":[{"name":"x","type":"log_count"}]}')
```

### Updating a rule

```bash
# ✅ CORRECT — get → edit → update
oodle log-metrics get lm_123 -o json > rule.json
jq '.labels = [{"name":"service","valueExtractor":{"field":"service"}},{"name":"env","valueExtractor":{"field":"env"}}]' rule.json > rule.new.json
oodle log-metrics update lm_123 -f rule.new.json

# ❌ WRONG — partial payload removes existing fields (and has no `id`, so it is rejected)
oodle log-metrics update lm_123 -f <(echo '{"labels":[{"name":"service","valueExtractor":{"field":"service"}}]}')
```

### Deleting a rule

```bash
# ✅ CORRECT — verify and delete
oodle log-metrics get lm_123 -o json > /dev/null
oodle log-metrics delete lm_123 --force

# ❌ WRONG — speculative delete by name match
oodle log-metrics delete "$(oodle log-metrics list | grep errors | awk '{print $1}')" --force
```

### Alerting on log lines

A monitor runs PromQL, not a log query. To alert on log lines, create a log metrics rule, then a monitor on the metric it emits (see the oodle-monitors skill). A rule reads the raw line text with a regex, so this works for plain-text lines, and you do not need a log transform. A label extracted from the line (user, tool, tenant) names the series in the alert.

Example: one alert per user whose credit ran out, from lines like:

```text
2026-10-07T10:12:03Z WARN credit exhausted for user=alice plan=pro
```

`credit-exhausted.json`:

```json
{
  "name": "credit-exhausted",
  "filter": {"all": [{"field": "message", "operator": "contains", "value": "credit exhausted"}]},
  "labels": [
    {"name": "user", "valueExtractor": {"field": "message", "regex": "user=(\\S+)"}}
  ],
  "metricDefinitions": [{"name": "credit_exhausted", "type": "log_count"}]
}
```

`credit-exhausted-monitor.json` (`op` 2 is `>`):

```json
{
  "name": "Credit exhausted",
  "promql_query": "sum by (user) (increase(oodle_logs_credit_exhausted[10m]))",
  "conditions": {"critical": {"op": 2, "value": 0}},
  "labels": {"team": "billing", "env": "prod"},
  "annotations": {"summary": "User {{ $labels.user }} ran out of credit"}
}
```

```bash
oodle log-metrics create -f credit-exhausted.json
oodle monitors create -f credit-exhausted-monitor.json
# Later: whose credit ran out in the last 24 hours
oodle metrics query --query 'sum by (user) (increase(oodle_logs_credit_exhausted[24h])) > 0' -o json
```

A label per user is fine here because the event is rare (see "Keep `labels` to 2–3 low-cardinality labels").

Example: a value inside the line, such as the duration in `tool Bash ok in 12345 ms`. Use one regex per value, each with one capture group, and a filter that requires the value:

```json
{
  "name": "tool-latency",
  "filter": {"all": [{"field": "message", "operator": "matches regex", "value": "tool \\S+ (?:ok|failed) in \\d+ ms"}]},
  "labels": [
    {"name": "tool", "valueExtractor": {"field": "message", "regex": "tool (\\S+) "}},
    {"name": "status", "valueExtractor": {"field": "message", "regex": " (ok|failed) in "}}
  ],
  "metricDefinitions": [
    {"name": "tool_calls", "type": "log_count"},
    {"name": "tool_duration_ms", "type": "histogram", "field": "message", "regex": " in (\\d+) ms"}
  ]
}
```

```promql
# p95 tool latency in ms, per tool
histogram_quantile(0.95, sum by (le, tool) (rate(oodle_logs_tool_duration_ms_bucket[5m])))
# failed calls per tool
sum by (tool) (increase(oodle_logs_tool_calls{status="failed"}[10m]))
```

## Best Practices

### Validate the `filter` against real log volume before creating the rule

A typo in the filter (`levl=eror`) creates a rule that silently emits zero data points.

```bash
# ✅ CORRECT — preview matching log volume in the log explorer first;
# only then run `oodle log-metrics create`
# (or run `oodle logs query` with the same condition and confirm count > 0)

# ❌ WRONG — create the rule, then notice the dashboard panel is empty next week
oodle log-metrics create -f rule.json
```

### Keep `labels` to 2–3 low-cardinality labels

Each label multiplies the time-series count. Avoid `request_id`, `user_id`, `trace_id`, `path` (with IDs). A user label is fine only on a rare event, such as credit refusals.

```bash
# ✅ CORRECT — bounded labels
"labels": [{"name": "service", "valueExtractor": {"field": "service"}}, {"name": "env", "valueExtractor": {"field": "env"}}]

# ❌ WRONG — `user_id` on every request blows up cardinality (one series per user)
"labels": [{"name": "service", "valueExtractor": {"field": "service"}}, {"name": "user_id", "valueExtractor": {"field": "user_id"}}]
```

### Always `get` before `update` to preserve fields

`update` replaces the document. Sending a single field nulls everything else.

```bash
# ✅ CORRECT
oodle log-metrics get lm_123 -o json > rule.json
jq '.filter.all += [{"field": "status", "operator": "matches regex", "value": "^5"}]' rule.json > rule.new.json
oodle log-metrics update lm_123 -f rule.new.json

# ❌ WRONG — clobbers labels and metricDefinitions
oodle log-metrics update lm_123 -f <(echo '{"filter":{"all":[{"field":"level","operator":"is","value":"error"}]}}')
```

### Use a stable, descriptive metric name

The emitted name is `oodle_logs_<name>`. A `<domain>_<measurement>` name makes the metric easy to find in dashboards and drop rules. Use only letters, digits, and underscores.

```bash
# ✅ CORRECT
"metricDefinitions": [{"name": "http_errors", "type": "log_count"}]

# ❌ WRONG — generic name collides with other rules
"metricDefinitions": [{"name": "errors", "type": "log_count"}]
```

## Failure Handling

| Error | Cause | Fix |
|-------|-------|-----|
| 401 Unauthorized | Invalid or missing API key | Run `oodle configure` or set `OODLE_API_KEY` |
| 404 Not Found | Log-metric rule ID does not exist | Verify with `oodle log-metrics list -o json` |
| connection refused | Wrong `OODLE_DEPLOYMENT` URL | Check `OODLE_DEPLOYMENT` env var |
| 400 `Match must have a valid Operator` | Operator is not `is`, `contains`, `matches regex`, or `exists` | Use one of the four; for "not equal", wrap the condition in `not` |
| 400 `Field ... must contain only alphanumeric characters and underscores` | Dotted field name in `field` | Use the top-level field and put the path in `jsonPath` |
| 400 `Regex ... is not a valid rust regex` | Regex syntax error, or a JSON escape is missing | Escape backslashes in JSON (`\\d`); no look-around |
| 400 `'logCount' metric definition has a Field, JSONPath, or Regex` | `log_count` has `field`, `jsonPath`, or `regex` | Remove them |
| 400 `Log Metrics ID in path and body don't match` | Body `id` is missing or differs from `<id>` | Run `get` and edit that output |
| Metric exists but has no data points | Filter doesn't match any logs, or no new lines since the rule was created | Re-test the filter in the log explorer or with `oodle logs query`; check field names match the log schema. Rules do not backfill |
| Label is empty / values are all 0 | The regex does not match, or group 1 is not a number | Test the regex on sample lines; make the filter require the value |
| Cardinality alarm in the UI | `labels` includes a high-cardinality field | Edit the rule to drop that label; old series age out at the regular retention |
| 429 Too Many Requests | Bulk rule creation | Add `--retries 3`, throttle to <10 creates per second |

## References

- [Oodle CLI repo](https://github.com/oodle-ai/oodle-cli)
- [Oodle docs](https://docs.oodle.ai)
