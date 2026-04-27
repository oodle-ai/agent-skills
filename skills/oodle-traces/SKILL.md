---
name: oodle-traces
description: Search and analyze Oodle traces by service, operation, duration, and error status.
metadata:
  version: "1.0.0"
  author: oodle-ai
  repository: https://github.com/oodle-ai/agent-skills
  tags: oodle,traces,spans,opentelemetry,apm
  globs: "**/*trace*,**/*span*,**/*otel*"
  alwaysApply: "false"
---

# Oodle Traces — Search and Analysis

This skill teaches the agent to search Oodle traces with the right service / time / duration filters and to fetch full trace detail by ID.

## Prerequisites

```bash
brew install oodle-ai/oodle/oodle
oodle configure
```

Confirm the traces endpoint works (use a recent time window):

```bash
oodle traces list --from -15m --to now -o json | jq 'length'
```

## Command Execution Order

Before running any oodle command:
1. Check whether a service name and approximate time window are already in context.
2. If not, run `oodle traces list --from -1h --to now -o json | jq -r '.[].service' | sort -u` to discover services that emitted traces recently.
3. Build the search with `--service`, `--from`, `--to` and at least one of `--error` / `--min-duration` / `--operation`.
4. To inspect a single trace, run `oodle traces get <trace-id> -o json`.
5. Do not run `oodle traces list` without a `--service` filter on production environments.

## Quick Reference

| Task | Command |
|------|---------|
| List traces in a service | `oodle traces list --service <svc> --from -1h --to now -o json` |
| List error traces only | `oodle traces list --service <svc> --error --from -1h -o json` |
| List slow traces (>500ms) | `oodle traces list --service <svc> --min-duration 500 --from -1h -o json` |
| Filter by operation | `oodle traces list --service <svc> --operation GET_/users --from -1h -o json` |
| Get full trace | `oodle traces get <trace-id> -o json` |

## Common Operations

### Searching for errors in the last hour

```bash
# ✅ CORRECT — scoped to one service, narrow time window, error filter on
oodle traces list --service api --error --from -1h --to now -o json

# ❌ WRONG — no service, no error filter, huge result set
oodle traces list --from -1h --to now -o json
```

### Searching for slow traces

```bash
# ✅ CORRECT — slow checkout traces in the last hour
oodle traces list --service checkout --min-duration 500 --from -1h --to now -o json

# ✅ CORRECT — narrow further by operation when you know the endpoint
oodle traces list --service checkout --operation POST_/cart --min-duration 500 --from -1h -o json

# ❌ WRONG — listing everything and sorting client-side wastes the API budget
oodle traces list --service checkout --from -1h -o json | jq 'sort_by(.durationMs) | reverse | .[0:10]'
```

### Fetching a full trace

```bash
# ✅ CORRECT — list to discover trace ids, then get the one of interest
TRACE_ID=$(oodle traces list --service api --error --from -15m -o json | jq -r '.[0].traceId')
oodle traces get "$TRACE_ID" -o json

# ❌ WRONG — `oodle traces get` without an id (it expects the id as a positional arg)
oodle traces get
```

### Time window flags

`--from` and `--to` accept the same formats as the rest of the CLI:

| Format | Example |
|--------|---------|
| Epoch seconds | `1731628800` |
| `now` | `now` |
| Relative duration | `-1h`, `-30m`, `-7d` |

```bash
# ✅ CORRECT
oodle traces list --service api --from -30m --to now -o json

# ❌ WRONG — human strings are not parsed
oodle traces list --service api --from "30 minutes ago"
```

## Best Practices

### Always specify `--service` to scope the search

Trace volume can be massive; an unfiltered list is expensive and rate-limited.

```bash
# ✅ CORRECT
oodle traces list --service api --from -1h -o json

# ❌ WRONG — full-tenant scan
oodle traces list --from -1h -o json
```

### Use `--error` to find failing traces, not client-side filtering

`--error` pushes the predicate to the server and returns matches faster.

```bash
# ✅ CORRECT
oodle traces list --service api --error --from -1h -o json

# ❌ WRONG — fetches every trace then filters client-side
oodle traces list --service api --from -1h -o json | jq '.[] | select(.status=="error")'
```

### Use `--min-duration` to find slow traces, not client-side sort

```bash
# ✅ CORRECT — server-side filter, returns ~the right number of rows
oodle traces list --service api --min-duration 500 --from -1h -o json

# ❌ WRONG — pulls everything, sorts on the client
oodle traces list --service api --from -1h -o json | jq 'sort_by(.durationMs) | reverse'
```

### Keep time windows ≤ 24h for `list`; widen by re-querying, not by raising `--to`

A 7-day list returns too many rows and risks timeouts. Page by hour.

```bash
# ✅ CORRECT — bounded window
oodle traces list --service api --error --from -1h --to now -o json

# ❌ WRONG — multi-day list
oodle traces list --service api --error --from -7d --to now -o json
```

## Failure Handling

| Error | Cause | Fix |
|-------|-------|-----|
| 401 Unauthorized | Invalid or missing API key | Run `oodle configure` or set `OODLE_API_KEY` |
| 404 Not Found | Trace ID does not exist or has aged out of retention | Re-list with a recent `--from` to find a current trace id |
| connection refused | Wrong `OODLE_DEPLOYMENT` URL | Check `OODLE_DEPLOYMENT` env var |
| Empty result | Wrong service name or time window outside retention | List recent services: `oodle traces list --from -1h -o json | jq -r '.[].service' | sort -u`; reduce the `--from` offset |
| Request times out | Time window too large or no service filter | Narrow `--from`/`--to`, add `--service`, add `--min-duration` or `--error` |
| 429 Too Many Requests | Concurrent trace fetches | Add `--retries 3`; throttle to <5 requests / second |

## References

- [Oodle CLI repo](https://github.com/oodle-ai/oodle-cli)
- [Oodle docs](https://docs.oodle.ai)
- [OpenTelemetry trace concepts](https://opentelemetry.io/docs/concepts/signals/traces/)
