---
name: oodle-monitors
description: Create, update, and manage Oodle monitors — alerting thresholds, query scoping, and best practices to avoid alert fatigue.
metadata:
  version: "1.1.0"
  author: oodle-ai
  repository: https://github.com/oodle-ai/agent-skills
  tags: oodle,monitors,alerting,alerts
  globs: "**/*monitor*,**/*alert*"
  alwaysApply: "false"
---

# Oodle Monitors — CRUD and Alerting

This skill teaches the agent to build, validate, and update Oodle monitors so that alerts are actionable, scoped, and free from flapping.

## Prerequisites

```bash
# Install + configure (see oodle-cli skill)
brew install oodle-ai/oodle/oodle
oodle configure
# or
export OODLE_API_KEY=<key>
export OODLE_INSTANCE=<instance>
export OODLE_DEPLOYMENT=<url>
```

Confirm authentication and that at least one monitor list call succeeds before creating new monitors:

```bash
oodle monitors list -o json | jq 'length'
```

## Command Execution Order

Before running any oodle command:
1. Check whether the required resource ID or name is already in context.
2. If not, run the discovery command (e.g., `oodle monitors list -o json`).
3. If the result is ambiguous, ask the user to confirm before proceeding.
4. Run the target command with the resolved ID.
5. Do not run speculative commands (e.g., do not `delete` without first `get`-ing the resource).

## Quick Reference

| Task | Command |
|------|---------|
| List all monitors | `oodle monitors list -o json` |
| Firing alerts | `oodle monitors triggers -o json` |
| Filter by labels | `oodle monitors list -o json \| jq '.[] \| select(.labels.env=="prod" and .labels.team=="platform")'` |
| Get one monitor | `oodle monitors get <id> -o json` |
| Create from file | `oodle monitors create -f monitor.json` |
| Update from file | `oodle monitors update <id> -f monitor.json` |
| Delete (CI) | `oodle monitors delete <id> --force` |

## Common Operations

### Listing monitors

```bash
# ✅ CORRECT — JSON output for scripting
oodle monitors list -o json

# ✅ CORRECT — list only firing alerts
oodle monitors triggers -o json

# ✅ CORRECT — narrow by labels for a specific team
oodle monitors list -o json | jq '.[] | select(.labels.env=="prod" and .labels.team=="platform")'

# ❌ WRONG — pulling everything then grepping
oodle monitors list | grep CPU
```

### Reading a monitor before changing it

```bash
# ✅ CORRECT — fetch full JSON, edit, then update
oodle monitors get mon_123 -o yaml > monitor.yaml
$EDITOR monitor.yaml
oodle monitors update mon_123 -f monitor.yaml

# ❌ WRONG — building update payload from memory; overwrites unrelated fields
oodle monitors update mon_123 -f <(echo '{"conditions":{"critical":{"op":2,"value":90}}}')
```

### Creating a monitor

A complete, valid monitor JSON:

```json
{
  "name": "High CPU on web servers",
  "promql_query": "avg by (host) (rate(node_cpu_seconds_total{env=\"prod\", service=\"api\", mode!=\"idle\"}[5m])) * 100",
  "conditions": {
    "critical": {"op": 2, "value": 80, "for": "5m", "keep_firing_for": "10m"},
    "warn": {"op": 2, "value": 60, "for": "5m", "keep_firing_for": "10m"}
  },
  "labels": {"team": "platform", "env": "prod"},
  "annotations": {
    "summary": "CPU above 80% on {{ $labels.host }}",
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
| `conditions.<severity>.for` | Optional. How long the condition must be true before the alert fires. |
| `conditions.<severity>.keep_firing_for` | Optional. How long the alert keeps firing after the condition clears. |
| `conditions.no_data` | Fires when the query returns no data. It must be `{"op": 0, "value": 1}`. |
| `labels` | Labels on every alert of the monitor. Notification policies route on them. |
| `annotations` | Text on the alert. Templates such as `{{ $labels.host }}` and `{{ $value }}` work. |
| `notifications` | Optional routing to notifiers (see the oodle-alerting skill). |

```bash
# ✅ CORRECT
oodle monitors create -f monitor.json

# ❌ WRONG — no `conditions`, monitor will be rejected
oodle monitors create -f <(echo '{"name":"x","promql_query":"up"}')
```

### Updating a monitor

```bash
# ✅ CORRECT — get → edit → update
oodle monitors get mon_123 -o json > monitor.json
jq '.conditions.critical.value = 85' monitor.json > monitor.new.json
oodle monitors update mon_123 -f monitor.new.json

# ❌ WRONG — sending only the changed field; missing fields become null
oodle monitors update mon_123 -f <(echo '{"conditions":{"critical":{"op":2,"value":85}}}')
```

### Deleting a monitor

```bash
# ✅ CORRECT — verify first, then delete
oodle monitors get mon_123 -o json > /dev/null
oodle monitors delete mon_123 --force

# ❌ WRONG — speculative delete by name match
oodle monitors delete "$(oodle monitors list | grep CPU | head -1 | awk '{print $1}')" --force
```

### Alerting on logs and traces

Monitors run PromQL only. They cannot run a log query or a TraceQL query. Alert on the metrics that logs and traces produce:

| Source | Metric | Skill |
|--------|--------|-------|
| Log lines (plain text or JSON) | `oodle_logs_<name>`, from a log metrics rule that counts lines or extracts a value with a regex | oodle-log-metrics |
| Trace spans | `oodle_trace_metrics` | oodle-traces |
| GenAI spans | `oodle_genai_*` | oodle-traces |

```json
{
  "name": "Credit exhausted",
  "promql_query": "sum by (user) (increase(oodle_logs_credit_exhausted[10m]))",
  "conditions": {"critical": {"op": 2, "value": 0}},
  "labels": {"team": "billing", "env": "prod"},
  "annotations": {"summary": "User {{ $labels.user }} ran out of credit"}
}
```

The monitor fires once for each series that crosses the threshold. Keep the label you need in `by (...)` so that the alert names the user.

## Best Practices

### Use a `[5m]` (or longer) range and a `for`, not `[1m]`

Short windows cause alert flapping on normal traffic spikes. Use `[5m]` and `"for": "5m"` minimum for production alerts, `[15m]` for noisy metrics.

```bash
# ✅ CORRECT
"promql_query": "avg by (host) (rate(node_cpu_seconds_total{env=\"prod\", mode!=\"idle\"}[5m])) * 100"

# ❌ WRONG — flaps on every brief spike
"promql_query": "avg by (host) (rate(node_cpu_seconds_total{env=\"prod\", mode!=\"idle\"}[1m])) * 100"
```

### Scope queries with explicit labels — never leave the selector empty

A selector with no label matchers matches every series of the metric and produces alerts for resources you don't own.

```bash
# ✅ CORRECT — scoped to a specific env + service
"promql_query": "avg by (host) (rate(node_cpu_seconds_total{env=\"prod\", service=\"api\", mode!=\"idle\"}[5m])) * 100"

# ❌ WRONG — alerts on every host in the org
"promql_query": "avg by (host) (rate(node_cpu_seconds_total[5m])) * 100"
```

### Always set `keep_firing_for` on the conditions

There are no recovery thresholds. Without `keep_firing_for` the alert resolves as soon as the value drops below the threshold — small oscillations make it resolve and fire again and again.

```bash
# ✅ CORRECT — the alert stays open 10 minutes after the value recovers
"conditions": {"critical": {"op": 2, "value": 80, "keep_firing_for": "10m"}, "warn": {"op": 2, "value": 60, "keep_firing_for": "10m"}}

# ❌ WRONG — no keep_firing_for; the monitor flaps around the threshold
"conditions": {"critical": {"op": 2, "value": 80}, "warn": {"op": 2, "value": 60}}
```

### Put a runbook URL in `annotations` and route with labels

Alerts without an action are noise. Every monitor must answer "what do I do?" (the annotations) and "who is paged?" (the labels that notification policies route on, or `notifications`; see the oodle-alerting skill).

```bash
# ✅ CORRECT
"annotations": {"summary": "CPU above 80% on {{ $labels.host }} (env=prod, service=api)", "runbook": "https://runbooks.example.com/cpu"}

# ❌ WRONG — no actionable content
"annotations": {"summary": "CPU is high"}
```

### Tag every monitor with at least `team` and `env` labels

Labels are how notification policies route alerts and how you filter `oodle monitors list -o json` with `jq`.

```bash
# ✅ CORRECT
"labels": {"team": "platform", "env": "prod", "service": "api"}

# ❌ WRONG
"labels": {}
```

## Failure Handling

| Error | Cause | Fix |
|-------|-------|-----|
| 401 Unauthorized | Invalid or missing API key | Run `oodle configure` or set `OODLE_API_KEY` |
| 404 Not Found | Monitor ID does not exist | Verify with `oodle monitors list -o json` |
| connection refused | Wrong `OODLE_DEPLOYMENT` URL | Check `OODLE_DEPLOYMENT` env var |
| `invalid promql query` | PromQL query has a syntax error | Test the query with `oodle metrics query --query '<promql>'` |
| `missing conditions` | No `critical`, `warn`, or `no_data` condition | Add at least one condition |
| `no data condition must have op:== and value:1` | Wrong `no_data` shape | Use `"no_data": {"op": 0, "value": 1}` |
| Alert never fires | Query returns no data, or threshold is unreachable | Run `oodle metrics names -o json \| jq -r '.[] \| select(startswith("<metric>"))'` to confirm the metric exists; lower the threshold temporarily and re-check |
| Too many alerts (flapping) | Range too short, no `for` or `keep_firing_for` | Increase the range to `[5m]` or `[15m]`; set `for` and `keep_firing_for` |
| `no data` alerts | Agent not reporting, or label filter excludes all hosts | Verify the agent is alive (`oodle metrics query --query 'up' -o json`); widen the label filter to confirm any series match |
| 429 Too Many Requests | Bulk monitor creation hit rate limit | Add `--retries 3`, throttle to <10 creates per second |

## References

- [Oodle CLI repo](https://github.com/oodle-ai/oodle-cli)
- [Oodle docs](https://docs.oodle.ai)
