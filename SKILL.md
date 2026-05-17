---
name: oodle
description: Oodle observability platform skills. Covers CLI usage, monitors, dashboards, alerting, metrics, traces, log metrics, synthetic monitors, drop rules, metric/log queries, integration onboarding, and documentation research.
metadata:
  version: "1.0.0"
  author: oodle-ai
  repository: https://github.com/oodle-ai/agent-skills
  tags: oodle,observability,monitoring,alerting,metrics,traces,docs
  alwaysApply: "false"
---

# Oodle Agent Skills

Skills for the Oodle observability platform CLI. Install with `oodle skills install` or `npx skills add`.

## Available Skills

| Skill | Description |
|-------|-------------|
| oodle-cli | Core CLI usage, auth, output formats, and common patterns |
| oodle-monitors | Monitor CRUD, alerting best practices, threshold configuration |
| oodle-dashboards | Dashboard and folder management |
| oodle-alerting | Notifiers, notification policies, and muting rules |
| oodle-metrics | Metric queries, label discovery, time ranges |
| oodle-traces | Trace search and span queries |
| oodle-log-metrics | Log-based metric rules |
| oodle-synthetic | Synthetic monitor management |
| oodle-metrics-query | PromQL instant and range queries |
| oodle-logs | Log search and index pattern discovery |
| oodle-drop-rules | Cost control via metric drop rules |
| oodle-onboarding | Integration onboarding — setup specs, step-by-step installation |
| oodle-docs | Documentation research workflow — read llms.txt and full docs page files |

## Install

### Primary: oodle CLI (recommended)
```bash
oodle skills install
```

### Claude Code
```bash
npx skills add oodle-ai/agent-skills -y
```

### Cursor
```bash
npx skills add oodle-ai/agent-skills --target-agent cursor -y
```

### Gemini CLI / Codex / Windsurf
```bash
npx skills add oodle-ai/agent-skills -y
```

## Quick Reference

| Task | Command |
|------|---------|
| List monitors | `oodle monitors list -o json` |
| Create monitor | `oodle monitors create -f monitor.json` |
| List dashboards | `oodle dashboards list -o json` |
| Search traces | `oodle traces list --service api --from -1h` |
| Query metrics | `oodle metrics list --match "http_requests"` |
| Instant PromQL query | `oodle metrics query --query "sum(up)" -o json` |
| Range PromQL query | `oodle metrics query-range --query "up" --start -1h --end now --step 60s -o json` |
| List log index patterns | `oodle logs index-patterns -o json` |
| Search logs | `oodle logs query -f query.ndjson -o json` |
| List notifiers | `oodle notifiers list -o json` |
| List drop rules | `oodle drop-rules list -o json` |
| List integrations | `oodle integrations list -o json` |
| Get integration setup spec | `oodle integrations get-setup-spec <type> -o json` |
| Read Oodle docs index | `curl -fsSL https://docs.oodle.ai/llms.txt` |
