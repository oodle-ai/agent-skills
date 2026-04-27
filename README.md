# Oodle Agent Skills

Public repository of agent skills for the [Oodle](https://oodle.ai) observability platform. Each skill is a structured `SKILL.md` document that teaches AI coding assistants (Claude Code, Cursor, Gemini CLI, Codex, Windsurf, and the Oodle CLI itself) how to drive the [`oodle` CLI](https://github.com/oodle-ai/oodle-cli) correctly — with prescriptive rules, executable examples, and clear failure-handling guidance.

These skills are designed so that an AI assistant can:

- Discover and operate Oodle resources (monitors, dashboards, notifiers, traces, metrics, synthetic monitors, drop rules, log metrics) with a single, predictable workflow.
- Avoid common mistakes such as alert fatigue, accidental dashboard overwrites, untested notifiers, and runaway metric ingestion costs.
- Produce output that other tools and scripts can consume (`-o json`) and run safely in CI (`--force`, `--retries N`).

## Available Skills

| Skill | Description |
|-------|-------------|
| [oodle-cli](skills/oodle-cli/SKILL.md) | Core CLI usage — auth, output formats, time flags, file input, common patterns |
| [oodle-monitors](skills/oodle-monitors/SKILL.md) | Monitor CRUD, alerting thresholds, query scoping, runbook links |
| [oodle-dashboards](skills/oodle-dashboards/SKILL.md) | Dashboard and folder management with safe-update / safe-delete patterns |
| [oodle-alerting](skills/oodle-alerting/SKILL.md) | Notifiers, notification policies, muting rules |
| [oodle-metrics](skills/oodle-metrics/SKILL.md) | Metric queries, label discovery, PromQL building |
| [oodle-traces](skills/oodle-traces/SKILL.md) | Trace search by service, operation, duration, error |
| [oodle-log-metrics](skills/oodle-log-metrics/SKILL.md) | Log-based metric rules — filters and groupBy |
| [oodle-synthetic](skills/oodle-synthetic/SKILL.md) | HTTP/TCP synthetic monitors with assertions |
| [oodle-drop-rules](skills/oodle-drop-rules/SKILL.md) | Metric drop / sample rules for ingestion cost control |

## Install

### Primary: Oodle CLI (recommended)

The Oodle CLI ships its own skills installer that places every skill in the right location for the AI agent that's currently active in your shell.

```bash
oodle skills install
```

### Claude Code

```bash
npx skills add oodle-ai/agent-skills -y
```

This installs the skills under `~/.claude/skills/` and registers the plugin manifest from `.claude-plugin/plugin.json`.

### Cursor

```bash
npx skills add oodle-ai/agent-skills --target-agent cursor -y
```

This installs the skills into Cursor's rules directory and uses `.cursor-plugin/plugin.json` for plugin metadata.

### Gemini CLI

```bash
npx skills add oodle-ai/agent-skills -y
```

The `gemini-extension.json` manifest at the repo root is picked up automatically.

### Codex

```bash
npx skills add oodle-ai/agent-skills -y
```

### Windsurf

```bash
npx skills add oodle-ai/agent-skills -y
```

### Manual install

Clone the repo and copy the `skills/` directory into your agent's skills location.

```bash
git clone https://github.com/oodle-ai/agent-skills.git
cp -r agent-skills/skills/* ~/.<agent>/skills/
```

## Quick Reference

| Task | Command |
|------|---------|
| Configure auth | `oodle configure` |
| List monitors | `oodle monitors list -o json` |
| Get monitor | `oodle monitors get <id> -o json` |
| Create monitor | `oodle monitors create -f monitor.json` |
| Update monitor | `oodle monitors update <id> -f monitor.json` |
| Delete monitor | `oodle monitors delete <id> --force` |
| List dashboards | `oodle dashboards list -o json` |
| Create dashboard | `oodle dashboards create -f dashboard.json` |
| List folders | `oodle folders list -o json` |
| List notifiers | `oodle notifiers list -o json` |
| List notification policies | `oodle notification-policies list -o json` |
| List muting rules | `oodle muting-rules list -o json` |
| Search metrics | `oodle metrics list --match "http_requests"` |
| List metric labels | `oodle metrics labels http_requests_total` |
| List label values | `oodle metrics label-values http_requests_total service` |
| Search traces | `oodle traces list --service api --from -1h` |
| Get trace | `oodle traces get <trace-id> -o json` |
| List log metrics | `oodle log-metrics list -o json` |
| List synthetic monitors | `oodle synthetic-monitors list -o json` |
| List drop rules | `oodle drop-rules list -o json` |

## Authoring Conventions

Every skill in this repo follows the same canonical structure:

1. YAML frontmatter (name, description, metadata)
2. Prerequisites
3. Command Execution Order
4. Quick Reference table
5. Common Operations with `✅ CORRECT` / `❌ WRONG` examples
6. Best Practices with prescriptive, testable rules
7. Failure Handling table
8. References

Rules in the skills are written prescriptively (`Run X` rather than `You can run X`) so that the agent treats them as hard constraints rather than suggestions.

## Links

- [Oodle CLI repo](https://github.com/oodle-ai/oodle-cli) — source of the `oodle` binary
- [Oodle docs](https://docs.oodle.ai) — product documentation
- [Issues](https://github.com/oodle-ai/agent-skills/issues) — report a problem or request a new skill

## License

[MIT](LICENSE)
