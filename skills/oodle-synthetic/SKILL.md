---
name: oodle-synthetic
description: Create and manage Oodle synthetic monitors — HTTP, TCP, and other health checks with expected status, body, and latency, alert conditions, and on-demand runs.
metadata:
  version: "1.1.0"
  author: oodle-ai
  repository: https://github.com/oodle-ai/agent-skills
  tags: oodle,synthetic,monitoring,health-checks
  globs: "**/*synthetic*"
  alwaysApply: "false"
---

# Oodle Synthetic Monitors — HTTP / TCP Checks

This skill teaches the agent to configure synthetic monitors that actually catch failures: expected status and latency, sane intervals, and update-without-overwrite. Setup, auth, `get`-before-`update`, safe delete, and common errors: see the oodle-cli skill.

## Workflow

1. Confirm the target URL or host and port with the user.
2. Test that it is reachable from the public internet (`curl -sSfI <url>`). A monitor on an unreachable target fails forever.
3. `oodle synthetic-monitors create -f monitor.json`.
4. Run it once with `oodle synthetic-monitors run <id>`, then check `last_result` in `oodle synthetic-monitors get <id> -o json`.

`oodle synthetic-monitors` has `list`, `get <id>`, `create -f`, `update <id> -f`, `delete <id>`, and `run <id>`.

## Schema

HTTP:

```json
{
  "name": "API health check (prod)",
  "enabled": true,
  "rule_type": "http",
  "rule_config": {
    "http": {
      "url": "https://api.example.com/health",
      "method": "GET",
      "expected_status_codes": [200],
      "max_response_time_ms": 2000,
      "expected_body": "ok",
      "follow_redirects": true
    }
  },
  "interval": "60s",
  "timeout": "10s",
  "conditions": {
    "critical": { "op": "<", "value": 1 }
  },
  "labels": { "team": "platform", "env": "prod" }
}
```

TCP: `"rule_type": "tcp"` and `"rule_config": {"tcp": {"host": "postgres.example.com", "port": 5432}}`, with the same other fields.

| Field | Meaning |
|-------|---------|
| `name` | Put the env in the name (`API health check (prod)`), so the person who gets the alert knows where to look. |
| `rule_type` | `http`, `tcp`, `ping`, `dns`, `ssl`, `traceroute`, or `multistep` |
| `rule_config.<rule_type>` | Config for that type. HTTP needs `url` and `method`; TCP needs `host` and `port`. |
| `expected_status_codes` | Status codes (`200`) or patterns (`"2XX"`) that pass. Always set it, or the check can pass on an error page. `excluded_status_codes` fails on a match. There is no `assertions` list. |
| `max_response_time_ms` | The check fails when the response is slower. A 200 after 30 seconds is still a failure for users. |
| `expected_body`, `expected_headers` | Content the response must have. |
| `interval` | Duration string, minimum `10s`. Use `60s` or more for most endpoints; short intervals on many endpoints multiply cost and noise. A plain number is not valid. |
| `timeout` | Duration string. At least `1s`, and not more than `interval`. |
| `conditions` | The check result is `1` (up) or `0` (down), so `{"op": "<", "value": 1}` alerts when a check fails. `op` is a string here: `>`, `>=`, `<`, `<=`, `==`, `!=`. |
| `agent_name` | Required for internal targets (private IPs, cluster-local names); routes the check through your agent. |
| `notifications`, `labels`, `annotations` | Routing and alert text, as for monitors. |

## Failure Handling

| Error | Cause | Fix |
|-------|-------|-----|
| Monitor always red | Target unreachable from the probe | `curl -I` from the public internet; check firewall, DNS, TLS cert |
| Monitor flaps | `interval` too short or `max_response_time_ms` too tight | Raise `interval` to `60s`; set `max_response_time_ms` to p95 plus headroom |
| `interval must be at least 10 seconds` / `timeout cannot exceed interval` | Wrong durations | Use duration strings such as `"60s"` and `"10s"` |
| `internal or kubernetes-local targets require an agent` | Private or cluster-local target | Set `agent_name`, or use a public URL |
| `rule_config` lost after an update | `update` with a partial body | Create it again from the last `get` output |
