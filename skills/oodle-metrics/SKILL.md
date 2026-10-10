---
name: oodle-metrics
description: Query Oodle metrics, discover labels and values, and build PromQL expressions using the label discovery workflow.
metadata:
  version: "1.1.0"
  author: oodle-ai
  repository: https://github.com/oodle-ai/agent-skills
  tags: oodle,metrics,promql,labels
  globs: "**/*metric*,**/*.promql,**/*.prom"
  alwaysApply: "false"
---

# Oodle Metrics — Discovery and Querying

This skill teaches the agent to find metric names, enumerate their labels, and build PromQL queries that return data on the first try. Setup, auth, time flag values, and common errors: see the oodle-cli skill. To run the queries, see the oodle-metrics-query skill.

## Quick Reference

| Task | Command |
|------|---------|
| Metric names with a prefix | `oodle metrics names -o json \| jq -r '.[] \| select(startswith("<prefix>"))'` |
| Metric names over a longer range | `oodle metrics names --start -24h --end now -o json` |
| Labels of a metric | `oodle metrics labels <name>` |
| Values of a label | `oodle metrics label-values <name> <label>` |

`names`, `labels`, and `label-values` read only series that have samples in the time range, and the default range is the last hour. A metric, label, or value with no samples in the range is not listed, even if it exists. Metrics from rare events (log metrics, error counters) are often sparse, so widen the range (for example `--start -7d`) before you decide that something does not exist. There is no `metrics list`, `metrics get`, `--match`, or `--limit`.

## The 3-step Discovery Workflow

Run these in order before you write PromQL for a metric you have not used before. Guessed label names give queries that return no data and look like a missing metric.

```bash
# 1 — find the metric name (filter with jq; the full list can be thousands of names)
oodle metrics names -o json | jq -r '.[] | select(startswith("http_requests"))'
# 2 — list its labels
oodle metrics labels http_requests_total
# 3 — list the values of a label
oodle metrics label-values http_requests_total service
```

```promql
# ✅ CORRECT — built from confirmed labels and values
sum by (service) (rate(http_requests_total{service="api",env="prod",status=~"5.."}[5m]))

# ❌ WRONG — guessed names; returns no data
sum by (svc) (rate(http_requests{app="api",environment="production",http_status=~"5.."}[5m]))
```

## Best Practices

- **Check cardinality before you group by a label:** `oodle metrics label-values <name> <label> | wc -l`. Grouping by a high-cardinality label such as `request_id` makes the query slow or time out.
- **Use `rate(...[5m])`, not `[1m]`.** `[1m]` rates are noisy on low-volume series and break when one scrape is missed.

## Failure Handling

| Error | Cause | Fix |
|-------|-------|-----|
| Metric, label, or value not found | It does not exist, or has no samples in the range | Retry with `--start -7d --end now` before you conclude it does not exist |
| Empty query result | Wrong label name or value | Run step 2 and step 3 again; fix the selector |
| Query timeout | High-cardinality `by (...)` | Remove high-cardinality labels; use a shorter time range |
| `parse error` | Broken PromQL syntax | Fix the expression; test it with `oodle metrics query` |

## References

- [PromQL reference](https://prometheus.io/docs/prometheus/latest/querying/basics/)
