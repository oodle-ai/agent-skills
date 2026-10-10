---
name: oodle-cli
description: Core Oodle CLI usage — auth, output formats, time flags, file input, and common patterns for all oodle commands.
metadata:
  version: "1.1.0"
  author: oodle-ai
  repository: https://github.com/oodle-ai/agent-skills
  tags: oodle,cli,observability
  globs: ""
  alwaysApply: "false"
---

# Oodle CLI — Core Usage

This skill teaches the agent to invoke the `oodle` CLI safely, with the right flags, time formats, and authentication for any subcommand. The other oodle skills refer to this one for setup, the rules that apply to every resource, and common errors.

## Prerequisites

```bash
# Install via Homebrew (macOS / Linux)
brew install oodle-ai/oodle/oodle

# Or via Go (any platform with Go 1.25+)
go install github.com/oodle-ai/oodle-cli/cmd/oodle@latest

# Configure auth (interactive — writes ~/.oodle/config.yaml)
oodle configure

# Or set env vars (preferred for CI / containers)
export OODLE_API_KEY=<your-api-key>
export OODLE_INSTANCE=<your-instance-id>
export OODLE_DEPLOYMENT=<your-deployment-url>   # e.g. https://app.oodle.ai

# Verify
oodle version
oodle monitors list -o json | head
```

In CI, take credentials from the secret store (`export OODLE_API_KEY="$CI_OODLE_API_KEY"`). Do not put a key on the command line, where it goes into shell history.

## Command Execution Order

Before running any oodle command:
1. Check whether the required resource ID or name is already in context.
2. If not, run the discovery command (e.g., `oodle <resource> list -o json`).
3. If the result is ambiguous, ask the user to confirm before proceeding.
4. Run the target command with the resolved ID.
5. Do not run speculative commands (e.g., do not `delete` without first `get`-ing the resource).

## Quick Reference

| Command | Purpose |
|---------|---------|
| `oodle monitors` | Manage metric monitors |
| `oodle dashboards` / `oodle folders` | Manage dashboards and dashboard folders |
| `oodle notifiers` | Manage notification channels |
| `oodle notification-policies` | Manage routing of alerts to notifiers |
| `oodle muting-rules` | Silence alerts during maintenance |
| `oodle metrics names` / `labels` / `label-values` | Discover metric names, labels, label values |
| `oodle metrics query` / `query-range` | Run a PromQL instant / range query |
| `oodle logs index-patterns` | List available log index patterns |
| `oodle logs query` | Search logs using OpenSearch Query DSL |
| `oodle traces list` / `get` | Search and fetch traces |
| `oodle traces labels` / `label-values` | Discover trace label names and values |
| `oodle traces traceql search '<query>'` | Find traces with a TraceQL filter (`--start`, `--end`, `--limit`). Alias: `oodle traces tql` |
| `oodle traces traceql metrics '<query>'` | TraceQL metrics: `rate()`, `count_over_time()`, `quantile_over_time()`, … (`--start`, `--end`, `--step`, `--exemplars`; `-o graph`, `-o stats`) |
| `oodle traces traceql tags` / `tag-values <tag>` | Discover TraceQL attribute names (`--scope resource\|span\|intrinsic\|all`) and values from the last hour (no `--start`/`--end`); `-q '<filter>'` limits to matching spans |
| `oodle log-metrics` | Manage log metrics rules (log lines → metrics; first step of a log alert) |
| `oodle synthetic-monitors` | Manage HTTP/TCP synthetic checks |
| `oodle drop-rules` | Manage metric drop rules |
| `oodle api-keys` / `oodle users` | Manage API keys and users |
| `oodle configure` | Interactive auth setup |
| `oodle skills` | Install and manage agent skills (this repo) |

## Global Flags

| Flag | Meaning |
|------|---------|
| `-o` | Output format: `table`, `json`, `yaml`, `csv`, `graph`, `stats`. Default `table` on a terminal. `graph` and `stats` are for time series, such as `oodle traces traceql metrics`. |
| `--force` | Skip the confirmation prompt of a destructive command. Required in a non-interactive pipeline. |
| `--retries` | Retries for transient 5xx and network errors. Default 3. Do not wrap the command in your own retry loop. |

Use `-o json` whenever a result goes to `jq`, a script, or another `oodle` command. Table columns can change between releases, so do not parse them with `awk` or `grep`. Use `-o yaml` to round-trip a resource into a `-f` file.

```bash
# ✅ CORRECT
ID=$(oodle monitors list -o json | jq -r '.[] | select(.name=="High CPU") | .id')

# ❌ WRONG — column positions in table output are not stable
ID=$(oodle monitors list | grep "High CPU" | awk '{print $1}')
```

## Time Flags

Time ranges use `--start` and `--end` (and `--time` for an instant query). There is no `--from` or `--to` flag. Every time flag of every command accepts the same values:

| Format | Example |
|--------|---------|
| `now` | `now` |
| Relative duration | `-30m`, `-1h`, `-7d` |
| RFC 3339 | `2026-10-07T10:00:00Z` |
| Epoch number in s, ms, µs, or ns | `1791367200`, `1791367200000` |

The CLI finds the epoch unit from the size of the number, so you do not need to know which unit the server uses. Prefer relative values.

Most commands default to `--start -1h --end now`. `oodle traces list` and `traces get` have no defaults: both flags are required.

```bash
# ✅ CORRECT
oodle traces list --service api --start -1h --end now -o json
oodle logs query -f q.ndjson --start 2026-10-07T00:00:00Z --end 2026-10-07T06:00:00Z -o json

# ❌ WRONG — --from/--to do not exist, and human strings are not parsed
oodle traces list --service api --from -1h --to now
oodle traces list --service api --start "1 hour ago" --end now
```

## File Input (`-f`)

`create` and `update` commands take `-f <file>`, JSON or YAML (found from the extension or the first byte). There is no inline `--body` flag.

## Rules for Every Resource

These apply to monitors, dashboards, notifiers, policies, muting rules, log metrics, drop rules, and synthetic monitors. The resource skills do not repeat them.

**`update` replaces the whole resource.** A partial body removes the fields you leave out (or fails when a required field or the `id` is missing). Always `get` → edit → `update`:

```bash
# ✅ CORRECT
oodle monitors get <id> -o json > m.json
jq '.conditions.critical.value = 85' m.json > m.new.json
oodle monitors update <id> -f m.new.json

# ❌ WRONG — only the changed field; the other fields are lost
oodle monitors update <id> -f <(echo '{"conditions":{"critical":{"op":2,"value":85}}}')
```

**Delete only a verified ID.** Resolve the ID once with `list -o json` and `jq`, confirm it with `get`, then delete. Never delete an ID found by `grep` on table output.

```bash
# ✅ CORRECT
oodle monitors get <id> -o json > /dev/null && oodle monitors delete <id> --force

# ❌ WRONG — speculative delete by name match
oodle monitors delete "$(oodle monitors list | grep CPU | head -1 | awk '{print $1}')" --force
```

**Capture an ID once and reuse it.** Do not resolve the name again at every step.

**Throttle bulk changes** to fewer than 10 creates per second.

## Common Errors

| Error | Cause | Fix |
|-------|-------|-----|
| 401 Unauthorized | Invalid or missing API key | Run `oodle configure` or set `OODLE_API_KEY` |
| 404 Not Found | Resource ID does not exist | Verify with `oodle <resource> list -o json` |
| connection refused | Wrong `OODLE_DEPLOYMENT` URL | Check `OODLE_DEPLOYMENT`, ensure no trailing slash, confirm network egress |
| 429 Too Many Requests | Rate limited | The CLI retries 3 times (`--retries`); back off 5–10s, send fewer requests per minute |
| `unknown command` / `unknown flag` | The command or flag does not exist, or the CLI is too old | Check `oodle <command> --help`; upgrade with `brew upgrade oodle` or `go install` |
| `error parsing -f file` | File is not valid JSON/YAML | Validate with `jq .` or `yq .` before retrying |

## References

- [Oodle CLI repo](https://github.com/oodle-ai/oodle-cli)
- [Oodle docs](https://docs.oodle.ai)
