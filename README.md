# Oodle Agent Skills

Public repository of agent skills for the [Oodle](https://oodle.ai) observability platform. Each skill is a structured `SKILL.md` document that teaches AI coding assistants (Claude Code, Cursor, Codex, GitHub Copilot, Factory.ai, Gemini CLI, Windsurf, and the Oodle CLI itself) how to drive the [`oodle` CLI](https://github.com/oodle-ai/oodle-cli) correctly — with prescriptive rules, executable examples, and clear failure-handling guidance.

These skills are designed so that an AI assistant can:

- Discover and operate Oodle resources (monitors, dashboards, notifiers, traces, metrics, synthetic monitors, drop rules, log metrics, integrations) and run PromQL/log queries with a single, predictable workflow.
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
| [oodle-metrics-query](skills/oodle-metrics-query/SKILL.md) | PromQL instant and range queries |
| [oodle-logs](skills/oodle-logs/SKILL.md) | Log search and index pattern discovery |
| [oodle-drop-rules](skills/oodle-drop-rules/SKILL.md) | Metric drop / sample rules for ingestion cost control |
| [oodle-onboarding](skills/oodle-onboarding/SKILL.md) | Integration onboarding — list integrations, fetch setup specs, execute step-by-step installation |
| [oodle-o11y-context](skills/oodle-o11y-context/SKILL.md) | Change-context manifest — the PR block that carries a change's observability intent across the SDLC |
| [oodle-o11y-review](skills/oodle-o11y-review/SKILL.md) | Left-side telemetry review — business context, cardinality, duplication, RED/USE/fan-out, alert-driven scrutiny |
| [oodle-triage](skills/oodle-triage/SKILL.md) | Right-side triage — single-alert context gathering and windowed auto-triage with fingerprint dedup |

## Install

### Native plugin marketplaces (recommended — auto-updating)

Several agents can install this repo as a plugin straight from GitHub and **auto-update** it — no file copying, no manual re-pulls. In every case the marketplace is named `oodle-ai` and the plugin is `oodle-agent-skills`, so the install target is `oodle-agent-skills@oodle-ai`.

**Claude Code** — reads `.claude-plugin/marketplace.json`:

```
/plugin marketplace add oodle-ai/agent-skills
/plugin install oodle-agent-skills@oodle-ai
```

Update on demand with `/plugin marketplace update oodle-ai` (plugins otherwise auto-update in the background).

**GitHub Copilot CLI** — also reads `.claude-plugin/marketplace.json` directly:

```
copilot plugin marketplace add oodle-ai/agent-skills
copilot plugin install oodle-agent-skills@oodle-ai
```

**Factory.ai (Droid)** — reads `.factory-plugin/marketplace.json`:

```
droid plugin marketplace add https://github.com/oodle-ai/agent-skills
droid plugin install oodle-agent-skills@oodle-ai
```

**Cursor** — reads `.cursor-plugin/marketplace.json`. Cursor onboards marketplaces through the GUI (no CLI add): Dashboard → **Plugins → Team Marketplaces → Add Marketplace → Import from Repo**, then paste `https://github.com/oodle-ai/agent-skills`. Requires the Cursor GitHub App on the repo; enable **Auto Refresh** to pull on push.

> **Codex** ships no separate plugin format — it reuses the Claude Code layout. Install Codex skills with `npx skills` below.

### Every agent: `npx skills`

The [`skills`](https://github.com/obra/skills) installer works across coding agents. Pick your `-a` agent slug (`-g` = global/user-level, `-y` = no prompts):

| Agent | Command |
|-------|---------|
| Claude Code | `npx skills add oodle-ai/agent-skills -a claude-code -y -g` |
| Cursor | `npx skills add oodle-ai/agent-skills -a cursor -y -g` |
| Codex | `npx skills add oodle-ai/agent-skills -a codex -y -g` |
| GitHub Copilot | `npx skills add oodle-ai/agent-skills -a copilot -y -g` |
| Factory.ai (Droid) | `npx skills add oodle-ai/agent-skills -a factory -y -g` |
| Gemini CLI | `npx skills add oodle-ai/agent-skills -a gemini-cli -y -g` |
| Windsurf | `npx skills add oodle-ai/agent-skills -a windsurf -y -g` |

Install to several agents at once with a comma-separated list, e.g. `-a claude-code,cursor,codex,copilot,factory`, or `-a '*'` for all supported agents.

Notes on what each agent picks up:

- **Claude Code** installs skills under `~/.claude/skills/` and registers `.claude-plugin/plugin.json`. Prefer the native marketplace above when you want auto-updates.
- **Cursor** installs into Cursor's rules directory using `.cursor-plugin/plugin.json` for plugin metadata.
- **Codex / Copilot / Factory.ai** install into each agent's skills/rules directory; they consume the same `SKILL.md` documents.
- **Gemini CLI** picks up the root `gemini-extension.json` manifest automatically.

### Oodle CLI

The Oodle CLI ships its own skills installer that places every skill in the right location for the AI agent that's currently active in your shell.

```bash
oodle skills install
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
| Instant PromQL query | `oodle metrics query --query "sum(up)" -o json` |
| Range PromQL query | `oodle metrics query-range --query "up" --start -1h --end now --step 60s -o json` |
| List log index patterns | `oodle logs index-patterns -o json` |
| Search logs | `oodle logs query -f query.ndjson -o json` |
| Search traces | `oodle traces list --service api --from -1h` |
| Get trace | `oodle traces get <trace-id> -o json` |
| List log metrics | `oodle log-metrics list -o json` |
| List synthetic monitors | `oodle synthetic-monitors list -o json` |
| List drop rules | `oodle drop-rules list -o json` |
| List integrations | `oodle integrations list -o json` |
| Get integration setup spec | `oodle integrations get-setup-spec <type> -o json` |

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
