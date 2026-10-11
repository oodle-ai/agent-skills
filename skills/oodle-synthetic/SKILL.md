---
name: oodle-synthetic
description: Create and manage Oodle synthetic monitors — HTTP and TCP health checks with expected status and latency, and configurable intervals.
metadata:
  version: "1.1.0"
  author: oodle-ai
  repository: https://github.com/oodle-ai/agent-skills
  tags: oodle,synthetic,monitoring,health-checks
  globs: "**/*synthetic*"
  alwaysApply: "false"
---

# Oodle Synthetic Monitors — HTTP / TCP Checks

This skill teaches the agent to configure synthetic monitors that actually catch failures: expected status and latency, sane intervals, and update-without-overwrite.

## Prerequisites

```bash
brew install oodle-ai/oodle/oodle
oodle configure
```

Confirm the synthetic-monitors endpoint works:

```bash
oodle synthetic-monitors list -o json | jq 'length'
```

## Command Execution Order

Before running any oodle command:
1. Check whether the target URL or hostport is already in context.
2. If not, ask the user to confirm the target.
3. Test the target manually (`curl -I <url>`) to confirm it is reachable from the public internet.
4. Run `oodle synthetic-monitors create -f monitor.json`.
5. After creation, verify the first run succeeded with `oodle synthetic-monitors get <id> -o json`.

## Quick Reference

| Task | Command |
|------|---------|
| List monitors | `oodle synthetic-monitors list -o json` |
| Get monitor | `oodle synthetic-monitors get <id> -o json` |
| Create monitor | `oodle synthetic-monitors create -f monitor.json` |
| Update monitor | `oodle synthetic-monitors update <id> -f monitor.json` |
| Delete monitor | `oodle synthetic-monitors delete <id> --force` |

## Common Operations

### Synthetic monitor schema (HTTP)

```json
{
  "name": "API health check",
  "enabled": true,
  "rule_type": "http",
  "rule_config": {
    "http": {
      "url": "https://api.example.com/health",
      "method": "GET",
      "expected_status_codes": [200],
      "max_response_time_ms": 2000
    }
  },
  "interval": "60s",
  "timeout": "10s",
  "conditions": {
    "critical": {"op": "<", "value": 1}
  }
}
```

| Field | Meaning |
|-------|---------|
| `rule_type` | `http` or `tcp` (also `ping`, `dns`, `ssl`, `traceroute`, `multistep`) |
| `rule_config.http` | `url` and `method` (HTTP) |
| `rule_config.tcp` | `host` and `port` (TCP) |
| `interval` | Duration string between probes (`"60s"`); minimum `10s`, recommended ≥ `60s` |
| `timeout` | Duration string; at least `1s`, and not more than `interval` |
| `expected_status_codes` | Status codes (`200`) or patterns (`"2XX"`) that pass. `excluded_status_codes` fails on a match |
| `max_response_time_ms` | The check fails when the response is slower |
| `expected_body`, `expected_headers` | Content the response must have (e.g. `"ok"`) |
| `conditions` | The check result is `1` (up) or `0` (down), so `{"op": "<", "value": 1}` alerts when a check fails. `op` is a string: `>`, `>=`, `<`, `<=`, `==`, `!=` |

### Synthetic monitor schema (TCP)

```json
{
  "name": "Postgres reachability",
  "enabled": true,
  "rule_type": "tcp",
  "rule_config": {
    "tcp": {"host": "postgres.example.com", "port": 5432}
  },
  "interval": "60s",
  "timeout": "5s",
  "conditions": {
    "critical": {"op": "<", "value": 1}
  }
}
```

### Creating a monitor

```bash
# ✅ CORRECT — manual reachability test, then create
curl -sSfI https://api.example.com/health
oodle synthetic-monitors create -f monitor.json

# ❌ WRONG — creating against an unreachable target leaves a permanently-failing monitor
oodle synthetic-monitors create -f monitor.json
```

### Updating a monitor

```bash
# ✅ CORRECT — get → edit → update preserves rule_config
oodle synthetic-monitors get syn_123 -o json > monitor.json
jq '.interval = "30s"' monitor.json > monitor.new.json
oodle synthetic-monitors update syn_123 -f monitor.new.json

# ❌ WRONG — partial payload removes rule_config
oodle synthetic-monitors update syn_123 -f <(echo '{"interval":"30s"}')
```

### Deleting a monitor

```bash
# ✅ CORRECT
oodle synthetic-monitors get syn_123 -o json > /dev/null
oodle synthetic-monitors delete syn_123 --force

# ❌ WRONG — name-grep delete
oodle synthetic-monitors delete "$(oodle synthetic-monitors list | grep health | awk '{print $1}')" --force
```

## Best Practices

### Use `interval: "60s"` (or higher) for non-critical endpoints

A 10-second interval on every endpoint multiplies cost and noise. Reserve short intervals for critical user-facing flows.

```bash
# ✅ CORRECT — 60s interval for a /health endpoint
"interval": "60s"

# ❌ WRONG — 10s on dozens of internal endpoints
"interval": "10s"
```

### Always set `expected_status_codes`

Without it, the check passes on any 2XX response, which may not be the response you expect.

```bash
# ✅ CORRECT
"expected_status_codes": [200], "max_response_time_ms": 2000

# ❌ WRONG — monitor "succeeds" on any 2XX response
"expected_status_codes": []
```

### Add `max_response_time_ms` to catch slow dependencies

A 200 response that takes 30 seconds is still a failure for users.

```bash
# ✅ CORRECT
"expected_status_codes": [200],
"max_response_time_ms": 2000

# ❌ WRONG — only checks status code, doesn't catch latency regressions
"expected_status_codes": [200]
```

### Always `get` before `update` to preserve `rule_config`

Update is a full-document replace.

```bash
# ✅ CORRECT
oodle synthetic-monitors get syn_123 -o json > m.json
jq '.rule_config.http.expected_body = "ok"' m.json > m.new.json
oodle synthetic-monitors update syn_123 -f m.new.json

# ❌ WRONG — drops `rule_config`
oodle synthetic-monitors update syn_123 -f <(echo '{"interval":"120s"}')
```

### Use a stable `name` per environment

Encode env in the name (`API health check (prod)`, `API health check (staging)`) so the alert recipient knows which env to investigate.

```bash
# ✅ CORRECT
"name": "API health check (prod)"

# ❌ WRONG
"name": "health"
```

## Failure Handling

| Error | Cause | Fix |
|-------|-------|-----|
| 401 Unauthorized | Invalid or missing API key | Run `oodle configure` or set `OODLE_API_KEY` |
| 404 Not Found | Synthetic monitor ID does not exist | Verify with `oodle synthetic-monitors list -o json` |
| connection refused | Wrong `OODLE_DEPLOYMENT` URL | Check `OODLE_DEPLOYMENT` env var |
| Monitor permanently red | Target unreachable from the probe egress | `curl -I` from the public internet; check firewall, DNS, TLS cert |
| Monitor flaps | `interval` too aggressive or `max_response_time_ms` too tight | Raise interval to `60s`; raise `max_response_time_ms` to p95+headroom |
| `interval must be at least 10 seconds` / `timeout cannot exceed interval` | Wrong durations, or a plain number | Use duration strings such as `"60s"` and `"10s"` |
| `rule_config` silently dropped | `update` was called with a partial payload | Re-create from the last `get` snapshot; always `get` → edit → `update` |
| 429 Too Many Requests | Many probes scheduled at the same second | Stagger by editing `interval` per monitor; use `--retries 3` for bulk operations |

## References

- [Oodle CLI repo](https://github.com/oodle-ai/oodle-cli)
- [Oodle docs](https://docs.oodle.ai)
