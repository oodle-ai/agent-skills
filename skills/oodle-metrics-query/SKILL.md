---
name: oodle-metrics-query
description: Execute PromQL instant and range queries against Oodle metrics using the Prometheus-compatible query API.
metadata:
  version: "1.1.0"
  author: oodle-ai
  repository: https://github.com/oodle-ai/agent-skills
  tags: oodle,metrics,query,promql,prometheus
  globs: "**/*query*,**/*.promql,**/*metric*"
  alwaysApply: "false"
---

# Oodle Metrics Query — PromQL Instant and Range Queries

This skill teaches the agent to execute PromQL instant and range queries against Oodle metrics using the Prometheus-compatible query API. Setup, auth, time flag values, and common errors: see the oodle-cli skill.

Confirm the metric names and labels first (see the oodle-metrics skill). A query on a metric that does not exist returns an empty result, not an error.

## Quick Reference

| Task | Command |
|------|---------|
| Instant query (now) | `oodle metrics query --query "sum(up)" -o json` |
| Instant query at a time | `oodle metrics query --query "up" --time -5m -o json` |
| Range query | `oodle metrics query-range --query "rate(http_requests_total[5m])" --start -1h --end now --step 60s -o json` |
| Range query, partial response | Add `--partial-response` |

Both commands need `--query`. `query-range` defaults to `--start -1h --end now --step 60s`.

## Choosing the Step

The step sets the resolution. Aim for a few hundred points per series:

```bash
# ✅ CORRECT — 1h at 60s (60 points); 24h at 5m (288 points)
oodle metrics query-range --query "up" --start -1h --end now --step 60s -o json
oodle metrics query-range --query "up" --start -1d --end now --step 5m -o json

# ❌ WRONG — 24h at 1s (86400 points)
oodle metrics query-range --query "up" --start -1d --end now --step 1s -o json
```

## Failure Handling

| Error | Cause | Fix |
|-------|-------|-----|
| 400 Bad Request | Invalid PromQL | Check the syntax |
| Empty result | Metric does not exist, or no data in the range | `oodle metrics names` to verify; widen the range |
| `required flag "query"` | Missing flag | Add `--query '<promql>'` |

## References

- [PromQL documentation](https://prometheus.io/docs/prometheus/latest/querying/basics/)
