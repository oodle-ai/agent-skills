---
name: oodle-monitors
description: Create, update, and manage Oodle monitors — PromQL alert queries, severity conditions, labels and annotations, and how to alert on logs and traces through metrics.
metadata:
  version: "1.1.0"
  author: oodle-ai
  repository: https://github.com/oodle-ai/agent-skills
  tags: oodle,monitors,alerting,alerts,promql
  globs: "**/*monitor*,**/*alert*"
  alwaysApply: "false"
---

# Oodle Monitors — CRUD and Alerting

This skill teaches the agent to build, validate, and update Oodle monitors so that alerts are actionable, scoped, and free from flapping. Setup, auth, `get`-before-`update`, safe delete, and common errors: see the oodle-cli skill.

A monitor runs one PromQL query and compares each result series with the thresholds in `conditions`. It fires once for each series that crosses a threshold. The labels of that series go into the alert.

## Quick Reference

| Task | Command |
|------|---------|
| List monitors | `oodle monitors list -o json` (no `--status` or `--labels` flag; filter with `jq '.[] \| select(.labels.team=="platform")'`) |
| Get one monitor / its state | `oodle monitors get <id> -o json` / `oodle monitors state <id> -o json` |
| Create / update | `oodle monitors create -f monitor.json` / `oodle monitors update <id> -f monitor.json` |
| Delete | `oodle monitors delete <id> --force` |
| Test the query first | `oodle metrics query --query '<promql>' -o json` |

## Monitor Schema

```json
{
  "name": "High CPU on api hosts",
  "promql_query": "avg by (host) (rate(node_cpu_seconds_total{env=\"prod\", service=\"api\", mode!=\"idle\"}[5m])) * 100",
  "conditions": {
    "critical": { "op": 2, "value": 90, "for": "5m" },
    "warn":     { "op": 2, "value": 80, "for": "10m" }
  },
  "labels": { "team": "platform", "env": "prod" },
  "annotations": {
    "summary": "CPU is {{ $value }}% on {{ $labels.host }}",
    "runbook": "https://runbooks.example.com/cpu"
  }
}
```

| Field | Meaning |
|-------|---------|
| `name` | Required. |
| `promql_query` | Required. A PromQL expression without the threshold. |
| `conditions` | Required. At least one of `critical`, `warn`, `no_data`. |
| `conditions.<severity>.op` | Comparison: `0` `==`, `1` `!=`, `2` `>`, `3` `>=`, `4` `<`, `5` `<=`. |
| `conditions.<severity>.value` | Threshold number. |
| `conditions.<severity>.for` | Optional. How long the condition must be true before the alert fires (`5m`). |
| `conditions.<severity>.keep_firing_for` | Optional. How long the alert keeps firing after the condition clears. |
| `conditions.no_data` | Fires when the query returns no data. It must be `{"op": 0, "value": 1}`. |
| `labels` | Labels on every alert of the monitor. Notification policies route on them. Templates such as `{{ $labels.host }}` work. |
| `annotations` | Text on the alert (summary, runbook). Templates such as `{{ $labels.user }}` and `{{ $value }}` work. |
| `notifications` | Optional. Routing: `[{"matchers": [], "notifiers": {"critical": ["<notifier-id>"]}}]`. See the oodle-alerting skill. |
| `interval`, `group_wait`, `group_interval`, `repeat_interval` | Optional timing. |

The schema is not the Datadog one: `type`, `query`, `message`, and `options.thresholds` are not fields.

## Alerting on Logs and Traces

Monitors run PromQL only. They cannot run TraceQL or log queries. Alert on the metrics that logs and traces produce:

| Source | Metric | Skill |
|--------|--------|-------|
| Log lines (plain text or JSON) | `oodle_logs_<name>`, from a log metrics rule that counts lines or extracts a value with a regex. No log transform is needed. | oodle-log-metrics |
| Trace spans (any service) | `oodle_trace_metrics` (span counts by `service_name`, `span_name`, `span_status`, `duration_ns_bucket`) | oodle-traces |
| GenAI spans | `oodle_genai_*` (tool calls, tool errors, tool latency, LLM calls, tokens, cost) | oodle-traces |

Example — one alert per user, from a log metrics rule:

```json
{
  "name": "Credit exhausted",
  "promql_query": "sum by (user) (increase(oodle_logs_credit_exhausted[10m]))",
  "conditions": { "critical": { "op": 2, "value": 0 } },
  "labels": { "team": "billing", "env": "prod" },
  "annotations": { "summary": "User {{ $labels.user }} ran out of credit" }
}
```

The oodle-log-metrics and oodle-traces skills have more examples.

## Best Practices

- **Use a range of 5 minutes or more and a `for`.** `[1m]` and no `for` flap on normal spikes. Use `[5m]` or more in `rate`/`increase`, and `"for": "5m"` or more in production.
- **Scope the query with label matchers** (`{env="prod", service="api"}`). With no matchers, the monitor alerts on every series of the metric, including resources you do not own.
- **Keep the labels you need with `by (...)`.** The alert carries only the labels that the result keeps. `sum(...)` with no `by` removes them all, so the alert cannot name the user or host.
- **Put a summary and a runbook URL in `annotations`.** An alert must tell the reader what to do.
- **Set at least `team` and `env` in `labels`.** Notification policies route on them, and `jq` finds monitors by them.

## Failure Handling

| Error | Cause | Fix |
|-------|-------|-----|
| `invalid promql query` | PromQL syntax error | Test with `oodle metrics query --query '<promql>'` |
| `missing conditions` | No `critical`, `warn`, or `no_data` | Add at least one condition |
| `no data condition must have op:== and value:1` | Wrong `no_data` shape | Use `"no_data": {"op": 0, "value": 1}` |
| Alert never fires | Query returns no data, or the threshold cannot be reached | Run the query; check the metric with `oodle metrics names` |
| Alert has no user/host label | The query removed the label | Add `by (<label>)` to the aggregation |
| Too many alerts (flapping) | Range too short, no `for` | Use `[5m]` or more and `"for": "5m"` |

## References

- [Oodle docs: Monitors](https://docs.oodle.ai/alerts/monitors)
