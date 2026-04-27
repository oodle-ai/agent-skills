---
name: oodle
description: Oodle observability platform CLI skills. Covers monitors, dashboards, alerting, metrics, traces, log metrics, synthetic monitors, and drop rules.
metadata:
  version: "1.0.0"
  author: oodle-ai
  repository: https://github.com/oodle-ai/agent-skills
  tags: oodle,observability,monitoring,alerting,metrics,traces
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
| oodle-drop-rules | Cost control via metric drop rules |

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
| List notifiers | `oodle notifiers list -o json` |
| List drop rules | `oodle drop-rules list -o json` |
