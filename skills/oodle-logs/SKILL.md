---
name: oodle-logs
description: Search and query log data using OpenSearch Query DSL, and discover available log index patterns.
metadata:
  version: "1.0.0"
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
oodle configure
```

Confirm the logs endpoint works:

```bash
oodle logs index-patterns -o json | jq 'length'
```

## Command Execution Order

Before running any log query:
1. Run `oodle logs index-patterns -o json` to discover available log index patterns.
2. Pick the appropriate index pattern `title` for your query.
3. Build the NDJSON query file using the discovered index pattern and OpenSearch Query DSL.
4. Do not guess index names — always confirm with `index-patterns` first.

## Quick Reference

| Task | Command |
|------|---------|
| List log index patterns | `oodle logs index-patterns -o json` |
| Search logs | `oodle logs query -f query.ndjson -o json` |

## Common Operations

### List log index patterns

Discover available log index patterns before building log queries. The returned `title` field is used as the `index` value in NDJSON query files.

```bash
# CORRECT — list all available index patterns
oodle logs index-patterns -o json

# CORRECT — extract just the pattern titles
oodle logs index-patterns -o json | jq '.[].title'

# CORRECT — use a discovered pattern in a log query
oodle logs index-patterns -o json | jq '.[].title'
# Then use the title in your NDJSON file:
# {"index": "oodle_internal_dev_logs"}
# {"query": {"match_all": {}}, "size": 10}
```

### Log query

Search log data using the OpenSearch-compatible multi-search API. The request body uses NDJSON format with two JSON objects separated by a newline: the first selects the index, the second contains the search query using OpenSearch Query DSL. Pass the body via `-f <file>`.

```bash
# CORRECT — create the NDJSON query file first
cat > query.ndjson <<'EOF'
{"index": "logs-*"}
{"query": {"match_all": {}}, "size": 10}
EOF
oodle logs query -f query.ndjson -o json

# CORRECT — search for specific log messages
cat > query.ndjson <<'EOF'
{"index": "logs-*"}
{"query": {"bool": {"must": [{"match": {"message": "error"}}]}}, "size": 50, "sort": [{"@timestamp": {"order": "desc"}}]}
EOF
oodle logs query -f query.ndjson -o json

# CORRECT — search with time range filter
cat > query.ndjson <<'EOF'
{"index": "logs-*"}
{"query": {"bool": {"must": [{"range": {"@timestamp": {"gte": "now-1h", "lte": "now"}}}]}}, "size": 100}
EOF
oodle logs query -f query.ndjson -o json

# WRONG — passing the query inline without -f
oodle logs query --body '{"index":"logs-*"}'

# WRONG — using JSON instead of NDJSON format
oodle logs query -f query.json   # file must be valid NDJSON (two lines)
```

## Best Practices

### Discover index patterns before querying

Always run `oodle logs index-patterns` to find the correct index name. Guessing index names may return empty results or errors.

```bash
# CORRECT — discover first, then query
oodle logs index-patterns -o json | jq '.[].title'
# Use the discovered title in your NDJSON file

# WRONG — guessing index names
cat > query.ndjson <<'EOF'
{"index": "my-logs"}
{"query": {"match_all": {}}, "size": 10}
EOF
oodle logs query -f query.ndjson -o json
```

### Use `-o json` for log query results

Log query responses contain nested JSON structures. Always use `-o json` and pipe through `jq` for extraction.

```bash
# CORRECT — extract log messages from the response
oodle logs query -f query.ndjson -o json | jq '.responses[].hits.hits[]._source.message'

# WRONG — using table output for complex nested log data
oodle logs query -f query.ndjson -o table
```

### Use index pattern fields to build precise queries

Each index pattern includes a `fields` array with field names and types. Use these to build targeted queries instead of `match_all`.

```bash
# CORRECT — check available fields first
oodle logs index-patterns -o json | jq '.[0].fields[].name'
# Then build a query using confirmed field names
```

## Failure Handling

| Error | Cause | Fix |
|-------|-------|-----|
| 401 Unauthorized | Invalid or missing API key | Run `oodle configure` or set `OODLE_API_KEY` |
| 400 Bad Request | Invalid NDJSON body or Query DSL | Validate the NDJSON file has exactly two lines; check OpenSearch Query DSL syntax |
| Empty result (not an error) | No matching logs in the index or time range | Widen the time range or check the index name with `oodle logs index-patterns` |
| connection refused | Wrong `OODLE_DEPLOYMENT` URL | Check `OODLE_DEPLOYMENT` env var, ensure no trailing slash |
| 429 Too Many Requests | Rate limited | Add `--retries 3`, back off, reduce query frequency |
| `required flag "file"` | Missing required flag for log query | Add `-f` flag with path to NDJSON file |

## References

- [Oodle CLI repo](https://github.com/oodle-ai/oodle-cli)
- [Oodle docs](https://docs.oodle.ai)
- [OpenSearch Query DSL](https://opensearch.org/docs/latest/query-dsl/)
