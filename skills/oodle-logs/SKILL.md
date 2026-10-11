---
name: oodle-logs
description: Search and aggregate log data using OpenSearch Query DSL (terms, date_histogram, percentiles, cardinality), discover log index patterns, and know when to use a log metrics rule instead (values inside plain-text lines, alerts).
metadata:
  version: "1.1.0"
  author: oodle-ai
  repository: https://github.com/oodle-ai/agent-skills
  tags: oodle,logs,search,opensearch,index-patterns
  globs: "**/*log*,**/*opensearch*,**/*.ndjson"
  alwaysApply: "false"
---

# Oodle Logs — Search and Index Pattern Discovery

This skill teaches the agent to search log data using the OpenSearch-compatible query DSL and to discover available log index patterns.

## Prerequisites

```bash
brew install oodle-ai/oodle/oodle
oodle configure   # or set OODLE_API_KEY, OODLE_INSTANCE, and OODLE_DEPLOYMENT

# Check access
oodle logs index-patterns -o json | jq '.[].title'
```

Other install methods, auth details, time flag values, and common errors: see the oodle-cli skill.

## Workflow

1. Run `oodle logs index-patterns -o json | jq '.[].title'`. Use a `title` as the `index` in the query file. Do not guess index names.
2. Get the field names and types: `oodle logs index-patterns -o json | jq '.[0].fields[].name'`.
3. Write a two-line NDJSON file and run `oodle logs query -f q.ndjson --start <start> --end <end> -o json`.
4. Read the hits with `jq '.responses[0].hits.hits[]._source'`, or the aggregations with `jq '.responses[0].aggregations'`. Use `-o json`; table output does not show nested data.

To alert on logs, use a log metrics rule plus a monitor (see the oodle-log-metrics skill).

## Log Query

The body is NDJSON with exactly two lines: the index header, then one search in OpenSearch Query DSL. Pass it with `-f`; there is no inline flag, and a pretty-printed JSON file is not valid.

- **`--start` defaults to `-1h` and `--end` to `now`.** For older data you must pass `--start`, or the query silently returns 0 results.
- **Do not put a `timestamp` range in the query body.** The CLI adds one from `--start`/`--end`, and a second one conflicts.
- One search per request. `size` must be less than 5000; to read more lines, split the time range.

```bash
# ✅ CORRECT
cat > q.ndjson <<'EOF'
{"index": "<index pattern title>"}
{"query": {"bool": {"must": [{"match_phrase": {"message": "OOMKilled"}}]}}, "size": 100, "sort": [{"timestamp": {"order": "desc"}}]}
EOF
oodle logs query -f q.ndjson --start -24h --end now -o json | jq -r '.responses[0].hits.hits[]._source.message'

# ❌ WRONG — no --start, so data older than 1 hour is not searched
oodle logs query -f q.ndjson -o json

# ❌ WRONG — timestamp range in the body
{"query": {"range": {"timestamp": {"gte": "now-6h"}}}}
```

### What the query DSL supports

| Clause | Use |
|--------|-----|
| `bool` (`must`, `filter`, `should`, `must_not`) | Combine clauses |
| `match_phrase` | Phrase in a text field, for example `{"match_phrase": {"message": "credit exhausted"}}` |
| `term`, `terms` | Exact value of a field |
| `range` | Numeric or time range on a real field |
| `exists` | Field is present |
| `query_string` | Terms, phrases, wildcards (`time*`), prefixes, `AND`/`OR`/`NOT`, and `field:>N` ranges on real fields |
| `ids`, `constant_score`, `match_all` | As in OpenSearch |

Aggregations (in an `aggs` block, nested as in OpenSearch): `terms`, `date_histogram` (`fixed_interval`, `calendar_interval`, or `interval`), `range`, `filters`, `avg`, `min`, `max`, `sum`, `percentiles`, `percentile_ranks`, `cardinality`.

**Not supported:** `regexp` queries, scripts (`script`, `script_score`, scripted fields), runtime fields, and grok. Plain `match` is not in the supported list; use `match_phrase`.

**So log search cannot filter on, or group by, a value that is only inside the text of a plain-text line.** For lines like `tool Bash ok in 12345 ms`, a query cannot select "more than 10000 ms" or group by the tool name. Say so, and offer an option. Do not tell the user that Oodle cannot do it:

1. **Log metrics rule** (oodle-log-metrics skill): extracts the value with a regex into a metric with labels. For counts, latency, and alerts on new logs.
2. **Log transform**: makes the value a real field at ingest, so `range`, `terms`, and `percentiles` work. New logs only.
3. **Past data**: find the lines with `match_phrase`, and extract the value on the client with `jq`.

### Aggregations

Set `"size": 0` when you need only the aggregation. The field names below are examples. Numeric aggregations need a numeric field, not text inside `message`.

```bash
# Error lines per service (top 20)
cat > q.ndjson <<'EOF'
{"index": "<index pattern title>"}
{"size": 0, "query": {"term": {"level": "error"}}, "aggs": {"by_service": {"terms": {"field": "service_name", "size": 20}}}}
EOF
oodle logs query -f q.ndjson --start -24h --end now -o json \
  | jq -r '.responses[0].aggregations.by_service.buckets[] | "\(.key)\t\(.doc_count)"'

# Other aggs blocks:
# lines per hour:   {"per_hour": {"date_histogram": {"field": "timestamp", "fixed_interval": "1h"}}}
# p95 per route:    {"by_route": {"terms": {"field": "route", "size": 10}, "aggs": {"lat": {"percentiles": {"field": "duration_ms", "percents": [50, 95, 99]}}}}}
# distinct users:   {"users": {"cardinality": {"field": "user_id"}}}
```

### Worked example: whose credit got exhausted in the last 24 hours?

First check if a log metrics rule already counts these lines: `oodle log-metrics list -o json`. If one does, read its metric, for example `oodle metrics query --query 'sum by (user) (increase(oodle_logs_credit_exhausted[24h])) > 0'`. A rule counts only lines that arrived after it was made, so use the logs below for older data.

```bash
cat > credit.ndjson <<'EOF'
{"index": "<index pattern title>"}
{"query": {"match_phrase": {"message": "credit exhausted"}}, "size": 1000, "sort": [{"timestamp": {"order": "desc"}}]}
EOF

# If the user is a real field (user_id, or log.user_id for a JSON attribute): use a terms aggregation on it.
# If the user is only inside the text ("... for user=alice ..."): extract it on the client.
oodle logs query -f credit.ndjson --start -24h --end now -o json \
  | jq -r '.responses[0].hits.hits[]._source.message | capture("user=(?<user>[^ ]+)").user' \
  | sort | uniq -c | sort -rn
```

If `hits.total` is more than `size`, split the time range (for example, four 6-hour queries) and combine the results. For an alert per user from now on, create a log metrics rule that extracts `user` with the regex `user=(\S+)`, and a monitor on `sum by (user) (increase(oodle_logs_credit_exhausted[10m]))`. The oodle-log-metrics skill has the JSON.

### Log query or log metrics rule?

| Need | Use |
|------|-----|
| Find lines, read events, investigate the past | `oodle logs query` |
| Count or group by a real field, over the past | `oodle logs query` with `aggs` |
| Filter or group by a value only inside plain text | Log metrics rule (new data), or `jq` on the client (past data) |
| Alert on log lines (count, rate, latency, per user) | Log metrics rule + monitor |
| Trend over weeks on a dashboard | Log metrics rule |

## Failure Handling

| Error | Cause | Fix |
|-------|-------|-----|
| 400 Bad Request | Invalid NDJSON or Query DSL | The file must have exactly two lines; check the DSL |
| 0 results | `--start` defaults to `-1h`, or wrong index or field | Pass `--start` (for example `-24h`); check `oodle logs index-patterns` |
| `required flag "file"` | No `-f` | Add `-f <file>.ndjson` |
| Error on `regexp`, `script`, or runtime field | Not supported | Use `match_phrase`, `query_string` wildcards, or a log metrics rule |
| Error on `size` | `size` is 5000 or more | Use a smaller `size` and split the time range |
| Error on multiple searches | More than one header/search pair | Run one query per search |

## References

- [OpenSearch Query DSL](https://opensearch.org/docs/latest/query-dsl/)
