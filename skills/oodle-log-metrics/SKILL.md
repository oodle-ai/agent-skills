---
name: oodle-log-metrics
description: Create and manage Oodle log metrics rules — turn log lines (plain text or JSON) into Prometheus metrics with filters, regex-extracted labels, and counters, gauges, or histograms. This is the first step of every alert on logs.
metadata:
  version: "1.1.0"
  author: oodle-ai
  repository: https://github.com/oodle-ai/agent-skills
  tags: oodle,log-metrics,logs,metrics,alerting
  globs: "**/*log-metric*,**/*logmetric*"
  alwaysApply: "false"
---

# Oodle Log Metrics — Rules, Extraction, and Log Alerts

This skill teaches the agent to turn log lines into metrics, and to alert on logs. A log metrics rule selects lines with a filter, extracts label values and numbers with regex or JSON paths, and emits Prometheus metrics. A monitor then alerts on those metrics with PromQL. Setup, auth, `get`-before-`update`, safe delete, and common errors: see the oodle-cli skill.

Facts to keep in mind:

- **You do not need a log transform.** A rule reads the raw line text with a regex. A log transform is only useful when you also want the value as a field in log search. Never tell the user that a log alert needs one.
- **A rule works on plain-text lines.** For example, it can read the user from `credit exhausted for user=alice` or the duration from `tool Bash ok in 12345 ms`.
- **A label can carry the user.** The monitor fires once per series, so a label extracted from the line (user, tool, tenant) names it in the alert.
- **No backfill.** A rule applies only to logs that arrive after you create it. For past data, use `oodle logs query` (see the oodle-logs skill).

## Workflow

1. Check that no rule counts these lines yet: `oodle log-metrics list -o json`. If one does, query its metric (`oodle_logs_<name>`) and stop.
2. Find sample lines with `oodle logs query` (see the oodle-logs skill). Note the field that holds the text (usually `message`) and the exact text shape.
3. Write the filter and the regexes, and test each regex on the samples (`grep -E`, `jq test()`). A regex that does not match gives an empty label and a value of 0, with no error.
4. Estimate cardinality: the distinct values of each extracted label, multiplied together.
5. `oodle log-metrics create -f rule.json`.
6. After new lines arrive, confirm the metric: `oodle metrics names -o json | jq -r '.[] | select(startswith("oodle_logs_"))'`.
7. To alert, create a monitor on the metric (see "Alerting on Logs").

`oodle log-metrics` (aliases `lm`, `logmetrics`) has `list`, `get <id>`, `create -f`, `update <id> -f`, and `delete <id>`.

## Rule Schema

```json
{
  "name": "credit-exhausted",
  "filter": { "all": [ { "field": "message", "operator": "contains", "value": "credit exhausted" } ] },
  "labels": [
    { "name": "env", "value": "prod" },
    { "name": "user", "valueExtractor": { "field": "message", "regex": "user=(\\S+)" } }
  ],
  "metricDefinitions": [
    { "name": "credit_exhausted", "type": "log_count" }
  ]
}
```

| Field | Meaning |
|-------|---------|
| `name` | Name of the rule. Required. |
| `filter` | Selects the log lines that the rule reads. If you omit it, the rule reads every line. |
| `labels` | Labels added to every metric of the rule. Each has a static `value` or a `valueExtractor`, not both. |
| `metricDefinitions` | One or more metrics to emit. At least one is required. |

### Filters

A filter is a tree. A condition has `field`, `operator`, `value`, and an optional `jsonPath`. Combine with `{"all": [...]}` (AND), `{"any": [...]}` (OR), and `{"not": <condition or filter>}`. You can nest them.

**Put a single condition inside an `all` list:** `{"all": [{"field": "message", "operator": "contains", "value": "timeout"}]}`. Newer servers also accept a condition inline or wrapped in `{"match": {...}}`, but older servers ignore the `match` wrapper and older CLI versions drop an inline condition. The `all` form works everywhere.

| Operator | Meaning |
|----------|---------|
| `is` | The field value equals `value` exactly. |
| `contains` | The field value contains `value`. The match ignores case. |
| `matches regex` | The field value matches the regex in `value` (Rust regex syntax). |
| `exists` | The field is present. No `value`. |

There is no `!=` operator. Use `{"not": {"field": ..., "operator": "is", "value": ...}}`.

### Fields

- `field` is a top-level log field name: letters, digits, and underscores only, not starting with a digit.
- Plain-text lines are usually in `message`. Parsed JSON attributes are usually inside `log`; read a nested value with `jsonPath`, for example `{"field": "log", "jsonPath": "user.id"}`. This works in conditions and extractors.
- Confirm field names with `oodle logs index-patterns -o json | jq '.[0].fields[].name'` and a sample from `oodle logs query`.

### Labels

| Label kind | Shape |
|------------|-------|
| Static | `{"name": "team", "value": "billing"}` |
| From a field | `{"name": "level", "valueExtractor": {"field": "level"}}` |
| From JSON | `{"name": "user", "valueExtractor": {"field": "log", "jsonPath": "user_id"}}` |
| From text with a regex | `{"name": "user", "valueExtractor": {"field": "message", "regex": "user=(\\S+)"}}` |

When the regex does not match a line, the label is empty for that line.

### Metric definitions

| `type` | What it emits | Value source |
|--------|---------------|--------------|
| `log_count` | Number of lines that match the filter. | None. Do not set `field`, `jsonPath`, or `regex`. |
| `counter` | Sum of the extracted values. | `field`, plus an optional `jsonPath` or `regex`. |
| `gauge` | Last extracted value. | `field`, plus an optional `jsonPath` or `regex`. |
| `histogram` | Distribution of the extracted values. | `field`, plus an optional `jsonPath` or `regex`. |

Value extraction rules:

- Use `jsonPath` or `regex`, not both.
- **The regex uses Rust syntax, and only capture group 1 is used** (the label value, or the number). Use `(?:...)` for groups that must not capture. Look-around is not supported.
- **When the regex does not match, or the text is not a number, the value is 0.** So make the filter select only lines that contain the number (a `matches regex` filter that requires it). If not, every other line adds a 0 sample.
- In JSON, escape each backslash: `\d` is `"\\d"`.

### Metric names in PromQL

- The emitted name is `oodle_logs_<metric name>`. The prefix is added for you; `get` shows the stored name with the prefix.
- `log_count` and `counter` keep that name with no `_total` suffix (`oodle_logs_credit_exhausted`). Query them with `increase(...)` or `rate(...)`. A `gauge` keeps the name; query it directly.
- A `histogram` emits `oodle_logs_<name>_bucket` (with `le`), `_sum`, and `_count`. The default bucket bounds are powers of two from about 0.1 to about 131072 (…, 4096, 8192, 16384, …), in the unit of your values. Check them with `oodle metrics label-values oodle_logs_<name>_bucket le`.
- Every series also has an `oodle_host` label. **Always aggregate with `sum by (...)`** over the labels you need, or one event shows as several series.

## Alerting on Logs

1. **Create a log metrics rule** that counts the lines, or extracts a value from them.
2. **Create a monitor** with PromQL on the emitted metric (see the oodle-monitors skill). Monitors run PromQL only.

The monitor fires once per series that crosses the threshold. Keep the label in the PromQL (`sum by (user)`) so the notification names the user.

### Example A — count matching lines, labelled by user

Sample lines:

```text
2026-10-07T10:12:03Z WARN credit exhausted for user=alice plan=pro
2026-10-07T10:14:41Z WARN credit exhausted for user=bob plan=free
```

`credit-exhausted.json`:

```json
{
  "name": "credit-exhausted",
  "filter": {
    "all": [ { "field": "message", "operator": "contains", "value": "credit exhausted" } ]
  },
  "labels": [
    { "name": "user", "valueExtractor": { "field": "message", "regex": "user=(\\S+)" } },
    { "name": "plan", "valueExtractor": { "field": "message", "regex": "plan=(\\S+)" } }
  ],
  "metricDefinitions": [
    { "name": "credit_exhausted", "type": "log_count" }
  ]
}
```

`credit-exhausted-monitor.json` (fires when a user runs out of credit; `op` 2 means `>`):

```json
{
  "name": "Credit exhausted",
  "promql_query": "sum by (user, plan) (increase(oodle_logs_credit_exhausted[10m]))",
  "conditions": { "critical": { "op": 2, "value": 0 } },
  "labels": { "team": "billing", "env": "prod" },
  "annotations": { "summary": "User {{ $labels.user }} ({{ $labels.plan }}) ran out of credit" }
}
```

```bash
oodle log-metrics create -f credit-exhausted.json
oodle monitors create -f credit-exhausted-monitor.json
```

A label per user is fine here because the event is rare. See "Cardinality".

### Example B — tool latency from lines like `tool Bash ok in 12345 ms`

One regex per value, each with one capture group:

```json
{
  "name": "tool-latency",
  "filter": {
    "all": [ { "field": "message", "operator": "matches regex", "value": "tool \\S+ (?:ok|failed) in \\d+ ms" } ]
  },
  "labels": [
    { "name": "tool", "valueExtractor": { "field": "message", "regex": "tool (\\S+) " } },
    { "name": "status", "valueExtractor": { "field": "message", "regex": " (ok|failed) in " } }
  ],
  "metricDefinitions": [
    { "name": "tool_calls", "type": "log_count" },
    { "name": "tool_duration_ms", "type": "histogram", "field": "message", "regex": " in (\\d+) ms" }
  ]
}
```

```promql
# p95 tool latency in ms, per tool
histogram_quantile(0.95, sum by (le, tool) (rate(oodle_logs_tool_duration_ms_bucket[5m])))
# failed calls per tool
sum by (tool) (increase(oodle_logs_tool_calls{status="failed"}[10m]))
```

Histogram buckets give only an approximate threshold (8192 or 16384 ms, not 10000 ms). **For an exact "over 10 s" count, add a `log_count` rule whose filter matches only slow calls**, for example the filter regex `tool \S+ (?:ok|failed) in \d{5,} ms` (10000 ms or more). The same pattern reads `key=value` lines: for `took_ms=1234`, use the value regex `took_ms=(\d+)`, and the filter regex `took_ms=\d{5,}\b` for "over 10 s".

## Cardinality

Each distinct label value makes a new series.

- ✅ OK: user id on a rare event (credit refusals, failed payments, a few lines a day).
- ❌ Not OK: user id, request id, trace id, or a URL path with ids on a high-volume line (every request, every tool call).

For high-volume lines, keep labels to a few low-cardinality values (`tool`, `status`, `env`), and find the user of a single event in log search.

## Updating a Rule

`update` replaces the whole rule, and the body must contain an `id` equal to the `<id>` argument. Use `get` → edit → `update`. An older CLI can show an empty filter in `get` for a rule with an inline condition; do not `update` from that output. Upgrade the CLI, or edit the rule in the Oodle UI.

```text
# ❌ WRONG — the schema is not a string expression with groupBy
{"name":"x","filter":"level=error AND service=api","groupBy":["service"],"metricName":"x"}
```

## Failure Handling

| Error | Cause | Fix |
|-------|-------|-----|
| 400 `Name is required` / `At least one MetricDefinition is required` | Missing required field | Add `name` and at least one `metricDefinitions` entry |
| 400 `Match must have a valid Operator` | Operator is not `is`, `contains`, `matches regex`, or `exists` | Use one of the four. For "not equal", wrap the condition in `not` |
| Rule counts every line | An older server or CLI dropped a `match` wrapper or an inline condition | Put the condition inside `{"all": [ ... ]}` |
| 400 `is not a valid rust regex` | Regex syntax error, or a JSON escape is missing | Escape backslashes in JSON (`\\d`); no look-around |
| 400 `Field ... must contain only alphanumeric characters and underscores` | Dotted field name in `field` | Use the top-level field and put the path in `jsonPath` |
| 400 `'logCount' metric definition has a Field` | `log_count` has `field`, `jsonPath`, or `regex` | Remove them |
| 400 on update: ID mismatch | Body `id` is missing or differs from `<id>` | Run `get` and edit that output |
| Metric has no data | No new matching lines since creation, or the filter does not match | Rules do not backfill. Check the filter with `oodle logs query` |
| Label is empty / values are all 0 | The regex does not match, or group 1 is not a number | Test the regex on sample lines; make the filter require the number |
| Too many series | High-cardinality extracted label | Remove the label, or narrow the filter to rare events |

## References

- [Oodle docs: Log Metrics](https://docs.oodle.ai/log-management/log-metrics)
- [Rust regex syntax](https://docs.rs/regex/latest/regex/#syntax)
