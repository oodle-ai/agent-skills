---
name: oodle-traces
description: Search and analyze Oodle traces by service, operation, duration, and error status; run TraceQL searches and TraceQL metrics; and alert on traces with the span metrics that every service emits (oodle_trace_metrics, oodle_genai_*).
metadata:
  version: "1.1.0"
  author: oodle-ai
  repository: https://github.com/oodle-ai/agent-skills
  tags: oodle,traces,spans,opentelemetry,apm,traceql,alerting
  globs: "**/*trace*,**/*span*,**/*otel*"
  alwaysApply: "false"
---

# Oodle Traces — Search, TraceQL, and Alerts

This skill teaches the agent to search Oodle traces, to run TraceQL from the CLI, and to alert on traces. Setup, auth, time flag values, and common errors: see the oodle-cli skill.

Facts to keep in mind:

- **Every service that sends spans has span metrics.** Oodle writes the PromQL metric `oodle_trace_metrics` from all spans. You do not configure it. Use it for trace dashboards and alerts. Never tell the user that Oodle has no span metrics or cannot alert on traces; check with `oodle metrics label-values oodle_trace_metrics service_name`.
- **GenAI spans have more metrics.** Spans with GenAI attributes also produce `oodle_genai_*` metrics (LLM calls, tool calls, tool errors, latency, tokens, cost).
- **Monitors run PromQL, not TraceQL.** Alert on `oodle_trace_metrics` or `oodle_genai_*`.
- **TraceQL runs from the CLI** with `oodle traces traceql search` and `oodle traces traceql metrics`.

## Workflow

1. Find the service: `oodle traces label-values service.name --start -1h --end now -o json`.
2. Search with `oodle traces list --service <svc>` and at least one of `--operation`, `--min-duration`, or `--tags`. For other conditions, use `oodle traces traceql search`.
3. For counts, rates, and latency over time, use `oodle traces traceql metrics` or PromQL on `oodle_trace_metrics`. Do not count traces from `list` output; `list` returns a limited sample.
4. To inspect one trace: `oodle traces get <trace-id> --start -1h --end now -o json`.
5. To alert, write PromQL on the span metrics and create a monitor (see "Alerting on Traces").

## `oodle traces list` and `get`

| Flag | Meaning |
|------|---------|
| `--start`, `--end` | **Required** for `list` and `get`. |
| `--service`, `--operation` | Service name; operation (span) name. |
| `--min-duration`, `--max-duration` | Trace duration with a unit: `500ms`, `2s`, `1m`. A plain number is rejected. |
| `--tags` | JSON map of label filters, all must match. Use names from `oodle traces labels`, for example `{"span_status":"Error"}`. |
| `--search` | Free-text search across trace data. |
| `--limit` | Maximum number of traces. |

There is no `--from`, `--to`, or `--error` flag. Always scope a list with `--service` and a short window; an unscoped list over days is slow and can time out. Filter on the server (`--tags`, `--min-duration`, TraceQL), not with `jq` on the client.

`list` returns `{"data": [ <trace> ], "total": N, ...}`. Each trace has `traceID`, `spans`, and `processes`. Each span has `operationName`, `startTime` (epoch µs), `duration` (µs), `tags`, and `processID`. The service of a span is `processes[<processID>].serviceName`.

```bash
# ✅ CORRECT — errors in one service, then the full trace
TRACE_ID=$(oodle traces list --service api --tags '{"span_status":"Error"}' --start -15m --end now --limit 1 -o json | jq -r '.data[0].traceID')
oodle traces get "$TRACE_ID" --start -1h --end now -o json

# ✅ CORRECT — slow traces, duration with a unit
oodle traces list --service checkout --min-duration 500ms --start -1h --end now -o json

# ❌ WRONG — --error and --from do not exist; get needs --start/--end
oodle traces list --service api --error --from -1h
oodle traces get "$TRACE_ID"
```

## TraceQL

`oodle traces traceql` has the alias `oodle traces tql`. Put the query in single quotes so the shell does not change `{`, `}`, `"`, or `!`.

```bash
# Find traces (filter only; a metrics query is rejected)
oodle traces traceql search '<filter>' [--start -1h] [--end now] [--limit 20]

# Time series from spans (needs a metrics function; a plain filter is rejected)
oodle traces traceql metrics '<filter> | <function>' [--start -1h] [--end now] [--step 5m] [--exemplars N] [-o graph|stats|json]

# Attribute names and values (reads only the last hour; no --start/--end)
oodle traces traceql tags [--scope resource|span|intrinsic|all] [-q '<filter>']
oodle traces traceql tag-values <tag> [-q '<filter>']
```

| Flag | Meaning |
|------|---------|
| `--start`, `--end` | `search` and `metrics`: default `-1h` and `now`. |
| `--limit` | `search`: maximum traces (default 20). |
| `--step` | `metrics`: `30s`, `5m`, `1h`, `1d`, or a number of seconds. Default `1m`, made larger to keep 300 points or fewer per series. |
| `--exemplars` | `metrics`: maximum exemplar traces. Shown only with `-o json` or `-o yaml`. |
| `-q`, `--query` | `tags`, `tag-values`: only look at spans that match this filter. |

Output: `search` prints one row per trace; `-o json` adds the matching spans and attributes. `metrics` prints one row per series (point count, last, min, max, average); `-o graph` draws a terminal chart, `-o stats` gives a summary, `-o json` gives all points.

### Supported syntax

| Part | Supported |
|------|-----------|
| Spanset filter | `{ <condition> }` |
| Attribute scopes | `resource.<name>`, `span.<name>`, and unscoped `.<name>` (matches span or resource) |
| Intrinsics | `name`, `status`, `duration`, `kind` |
| Comparison | `=` `!=` `=~` `!~` `>` `>=` `<` `<=` |
| Combine in one spanset | `&&`, `||`, `!( ... )` |
| Union of spansets | `{ A } || { B }` |
| Attribute present / absent | `span.foo != nil`, `span.foo = nil` |
| Literals | strings `"x"`, numbers, durations `10s` `250ms`, `status = error` / `ok` / `unset`, `kind = server` / `client` / `internal` / `producer` / `consumer` |
| Metrics functions | `rate()`, `count_over_time()`, `avg_over_time(<field>)`, `min_over_time(<field>)`, `max_over_time(<field>)`, `sum_over_time(<field>)`, `histogram_over_time(<field>)`, `quantile_over_time(<field>, 0.5, 0.95, ...)` |
| Grouping | `by (<attribute>, ...)` after a metrics function |

### Not supported yet

| Syntax | Use instead |
|--------|-------------|
| Scalar filters: `{ ... } \| count() > 2`, `\| avg(duration) > 1s` | `traceql metrics` with `count_over_time()` or `avg_over_time()`, and read the values |
| `{ A } && { B }` (two different spans of one trace) | One spanset `{ A && B }` when one span must match both |
| Structural operators `>>`, `<<`, `~` | Search for each span on its own, then compare trace ids |
| `parent.` and `link.` scopes | `span.` or `resource.` attributes |
| `rootName`, `traceDuration` | `name` and `duration` of a single span (for example, the root span by its name) |
| Bare existence: `{ span.foo }` | `{ span.foo != nil }` |

If a query fails with "not supported", rewrite it with the table above. Do not tell the user that TraceQL is not available.

### Examples

```bash
# Errors in one service
oodle traces traceql search '{ resource.service.name = "checkout" && status = error }' --start -1h --limit 20

# GenAI tool calls over 10 s
oodle traces traceql search '{ span.gen_ai.tool.name != nil && duration > 10s }' --start -24h --limit 50

# Error rate by service
oodle traces traceql metrics '{ status = error } | rate() by (resource.service.name)' --start -6h --step 5m

# p50 and p95 latency per route, as a terminal chart
oodle traces traceql metrics '{ kind = server } | quantile_over_time(duration, 0.5, 0.95) by (span.http.route)' --start -1h -o graph

# Error rate with 5 example traces, for drill-down
oodle traces traceql metrics '{ status = error } | rate()' --start -1h --exemplars 5 -o json

# Attribute names and values
oodle traces traceql tags --scope span
oodle traces traceql tag-values span.http.route -q '{ resource.service.name = "checkout" }'
```

## Alerting on Traces

### `oodle_trace_metrics`

Every span adds 1 to `oodle_trace_metrics`, as a count per minute. Query it with `increase(...)` or `rate(...)`.

| Label | Meaning |
|-------|---------|
| `service_name`, `span_name` | Service and span (operation) name |
| `span_status` | `Ok`, `Error`, or `Unset` |
| `duration_ns_bucket` | Lower bound of the span duration bucket, in nanoseconds |
| `is_root_span` | `true` for the root span of a trace |
| `is_first_span_of_service` | `true` for the entry span of a service (a request into the service) |
| `cluster`, `env`, `namespace` | Where the span came from |

A span is in the largest bucket whose lower bound is not more than its duration. The bounds are `0`, `100000` (0.1 ms), `200000`, `500000`, `1000000` (1 ms), then doubling `2000000` … `512000000`, then `1000000000` (1 s), `2000000000`, `4000000000`, `8000000000`, `16000000000`, `32000000000`, `64000000000`, `128000000000` (128 s and more). So "slower than 10 s" is not a bucket edge: use "8 s or more" or "16 s or more", or TraceQL for an exact count.

```promql
# Error ratio per service (entry spans)
sum by (service_name) (increase(oodle_trace_metrics{is_first_span_of_service="true", span_status="Error"}[5m]))
  / sum by (service_name) (increase(oodle_trace_metrics{is_first_span_of_service="true"}[5m]))

# Spans of 8 s or more, per service and span
sum by (service_name, span_name) (increase(oodle_trace_metrics{duration_ns_bucket=~"8000000000|16000000000|32000000000|64000000000|128000000000"}[10m]))
```

Example monitor (`op` 2 means `>`; see the oodle-monitors skill for the schema):

```json
{
  "name": "Checkout error ratio",
  "promql_query": "sum by (service_name) (increase(oodle_trace_metrics{service_name=\"checkout\", is_first_span_of_service=\"true\", span_status=\"Error\"}[5m])) / sum by (service_name) (increase(oodle_trace_metrics{service_name=\"checkout\", is_first_span_of_service=\"true\"}[5m]))",
  "conditions": { "critical": { "op": 2, "value": 0.05, "for": "10m" } },
  "labels": { "team": "payments", "env": "prod" },
  "annotations": { "summary": "{{ $labels.service_name }} error ratio is {{ $value }}" }
}
```

### `oodle_genai_*` (GenAI spans)

Spans with GenAI attributes (model calls, agent steps, tool calls) also produce these metrics. Query counters with `increase(...)` or `rate(...)`.

| Metric | What it counts | Main labels |
|--------|----------------|-------------|
| `oodle_genai_tool_call_total` | Tool calls | `tool_name`, `service_name`, `model`, `env` |
| `oodle_genai_tool_error_total` | Tool calls that failed | `tool_name`, `service_name`, `model`, `env` |
| `oodle_genai_tool_latency_seconds` | Tool call latency, cumulative histogram | `le`, `tool_name`, `service_name` |
| `oodle_genai_llm_call_total` | Direct model calls | `model`, `operation`, `status`, `service_name` |
| `oodle_genai_llm_call_latency_seconds` | Model call latency, cumulative histogram | `le`, `model`, `operation`, `service_name` |
| `oodle_genai_span_total` | All GenAI spans | `operation`, `span_name`, `status`, `model`, `service_name` |
| `oodle_genai_span_error_total` | GenAI spans with errors | `error_type`, `span_name`, `model`, `service_name` |
| `oodle_genai_generation_tokens_total` | Tokens | `type`, `model`, `service_name` |
| `oodle_genai_generation_cost_dollars` | Cost in US dollars | `model`, `service_name` |
| `oodle_genai_generation_ttft_seconds` | Time to first token, cumulative histogram | `le`, `model` |

Series also carry `agent_name` when the span reports an agent. The latency histograms have the `le` label on the metric itself; there is no `_bucket` suffix. The latency `le` values are `0.1`, `0.25`, `0.5`, `1`, `2`, `4`, `8`, `16`, `32`, `64`, `128`, `256`, `512`, `+Inf` (seconds).

```promql
# Tool error ratio per tool
sum by (tool_name) (increase(oodle_genai_tool_error_total[15m]))
  / sum by (tool_name) (increase(oodle_genai_tool_call_total[15m]))

# Tool calls over 8 s, per tool (all calls minus calls of 8 s or less)
sum by (tool_name) (increase(oodle_genai_tool_latency_seconds{le="+Inf"}[15m]))
  - sum by (tool_name) (increase(oodle_genai_tool_latency_seconds{le="8"}[15m]))

# p95 model call latency per model
histogram_quantile(0.95, sum by (le, model) (rate(oodle_genai_llm_call_latency_seconds[5m])))
```

Confirm the metric and label names before you create a monitor:

```bash
oodle metrics names -o json | jq -r '.[] | select(startswith("oodle_genai_") or . == "oodle_trace_metrics")'
oodle metrics label-values oodle_genai_tool_call_total tool_name
```

## Failure Handling

| Error | Cause | Fix |
|-------|-------|-----|
| `required flag(s) "end", "start" not set` | `list` and `get` need both flags | Add `--start -1h --end now` |
| `unknown flag: --from` / `--error` | Flags do not exist | Use `--start`/`--end`; use `--tags '{"span_status":"Error"}'` for errors |
| 400 on `--min-duration` | Duration has no unit | Use `500ms`, `2s` |
| 404 on `get` | Trace ID is wrong or past retention | List again with a recent `--start` |
| Empty result | Wrong service name or time window | `oodle traces label-values service.name --start -1h --end now` |
| `unknown trace label` | Label name is wrong | Run `oodle traces labels`; use the suggested name |
| TraceQL `... is not supported` | Syntax from "Not supported yet" | Rewrite with the replacements in that table |
| `search` rejects the query | It has a metrics function (`\| rate()` …) | Use `oodle traces traceql metrics` |
| `metrics` rejects the query | It has no metrics function | Add one, or use `oodle traces traceql search` |
| Request times out | Window too large or no service filter | Narrow `--start`/`--end`, add `--service`, `--min-duration`, or `--tags` |

## References

- [Oodle docs: Traces](https://docs.oodle.ai/traces)
- [TraceQL](https://grafana.com/docs/tempo/latest/traceql/)
