# Oodle Observability-in-SDLC Skills — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship three portable `SKILL.md` skills — `oodle-o11y-context`, `oodle-o11y-review`, `oodle-triage` — that embed observability judgment into the left (authoring/review) and right (production triage) of the SDLC, connected by a PR-carried change-context manifest.

**Architecture:** Pure `SKILL.md` docs (no hooks/commands/subagents/templates), matching the existing 12 skills. A tiny spine skill defines the `## Oodle Change Context` PR block; the review skill writes it, the triage skill reads it. Backend and issue-tracker are discovered at runtime by MCP tool *shape* (never a hard-coded server name); Oodle is the tight path, any vendor MCP the generic path.

**Tech Stack:** Markdown `SKILL.md` with YAML frontmatter (`name`, `description`, `metadata.{version,author,repository,tags,globs,alwaysApply}`). Validation via `skill-quality-reviewer` (structure/style) and the installed **skill-creator** plugin's Eval/Benchmark modes (behavioral eval against prompt scenarios). Reference source: `~/oodle/.claude/commands/oncall-debug.md`, `~/oodle/.claude/commands/oncall-triage.md`.

**Spec:** `docs/superpowers/specs/2026-07-14-oodle-o11y-sdlc-skills-design.md`

---

## File Structure

- Create: `skills/oodle-o11y-context/SKILL.md` — manifest format + relevance gate (spine). ~80 lines.
- Create: `skills/oodle-o11y-review/SKILL.md` — left: discovery preamble, four checks, output gating, handoffs. ~150 lines.
- Create: `skills/oodle-triage/SKILL.md` — right: discovery preamble, Mode A (context), Mode B (auto-triage), fingerprint format. ~180 lines.
- Create: `skills/oodle-o11y-review/evals.md` — eval scenarios for skill-creator (not a skill; a test fixture).
- Create: `skills/oodle-triage/evals.md` — eval scenarios for skill-creator.
- Modify: `README.md` — add the three skills to the "Available Skills" table.

No manifest edits: `.claude-plugin/plugin.json`, `.cursor-plugin/plugin.json`, `gemini-extension.json` are generic and auto-discover skill directories.

**Shared frontmatter template** (every new SKILL.md):

```yaml
---
name: <slug>
description: <one-line, third person, what+when>
metadata:
  version: "1.0.0"
  author: oodle-ai
  repository: https://github.com/oodle-ai/agent-skills
  tags: <comma,separated>
  globs: "<comma-separated globs or empty>"
  alwaysApply: "false"
---
```

**Skill-creator folded in:** after authoring each behavioral skill (review, triage), its `evals.md` is run through skill-creator's Eval mode (Executor runs the skill against each prompt, Grader scores against the rubric). Task 5 is a dedicated Eval/Improve pass. The spine skill (context) is validated structurally only (no behavior to eval).

---

## Task 1: Spine — `oodle-o11y-context`

**Files:**
- Create: `skills/oodle-o11y-context/SKILL.md`

- [ ] **Step 1: Write the SKILL.md**

Frontmatter: `name: oodle-o11y-context`, `description: "Defines the Oodle Change Context manifest — the structured PR block that carries a change's observability intent (type, touched services, telemetry added/changed, guarding alerts, watch items, gaps) from authoring through review into production triage. Referenced by oodle-o11y-review and oodle-triage."`, `tags: oodle,observability,sdlc,pr,change-context`, `globs: ""`.

Required sections (content from spec §Spine):
1. **Purpose** — one paragraph: the manifest is the connective tissue of the loop; every other o11y skill reads/writes it.
2. **The block** — the exact fenced `## Oodle Change Context` template with the six fields (`type`, `touches`, `telemetry` with `+/~/-` + purpose, `guarded-by`, `watch`, `gaps`), shown as a worked checkout example.
3. **Field reference** — a table: field → meaning → who fills it (author vs review) → why it matters downstream.
4. **Lifecycle** — the 4-step loop (author drafts → review enriches → merge → triage reads).
5. **Relevance gate** — when the block IS expected (touches instrumented/service code, or type perf/critical-path/bugfix) vs skipped (docs/trivial). The reviewer decides.
6. **Carrier** — PR description only in v1; read back via the GitHub/Linear MCP at triage. Note git-trailer/tracked-file as explicitly deferred.
7. **Stability contract** — the `## Oodle Change Context` header and field keys are stable so downstream agents match them; don't rename.

- [ ] **Step 2: Structural validation**

Invoke `skill-quality-reviewer` on `skills/oodle-o11y-context/SKILL.md`.
Expected: no critical structural findings; description within length; frontmatter valid. Fix any critical/important findings inline.

- [ ] **Step 3: Commit**

```bash
git add skills/oodle-o11y-context/SKILL.md
git commit -m "feat: add oodle-o11y-context skill (change-context manifest spine)"
```

---

## Task 2: Left — `oodle-o11y-review`

**Files:**
- Create: `skills/oodle-o11y-review/SKILL.md`
- Create: `skills/oodle-o11y-review/evals.md`

- [ ] **Step 1: Write evals.md first (the behavioral "tests")**

Concrete scenarios, each: `PROMPT` (a diff + ask to review telemetry) → `EXPECT` (what the skill should make the agent do/flag). Minimum five:
1. **High-card metric label** — diff adds `orders_total{order_id=...}` counter → EXPECT: flagged critical, recommend moving `order_id` to a span attribute / log field, hand off to `oodle-drop-rules`.
2. **Missing tenant context** — diff adds a span in tenant-scoped handler with no tenant attr → EXPECT: flagged, recommend tenant_id span attribute (high-card OK on spans).
3. **Duplicate telemetry** — diff adds a timer duplicating an existing histogram → EXPECT: flagged, recommend removal, manifest `-` line.
4. **N+1 / fan-out** — diff adds a loop issuing per-item DB calls, no batched/measured span → EXPECT: flagged as perf-coverage gap.
5. **Alert-driven scrutiny** — touched code sits under an existing latency SLO but lacks localizing spans → EXPECT: critical finding, `guarded-by` recorded in manifest, change marked higher-risk. Includes the case where NO backend MCP is present → EXPECT: checks 1–3 still run, check 4 explicitly degraded with a note.

Each scenario ends with a one-line grading rubric (what a pass looks like).

- [ ] **Step 2: Write the SKILL.md**

Frontmatter: `name: oodle-o11y-review`, `description: "Reviews a code change's telemetry during authoring/review — checks business-context attributes and cardinality placement, telemetry duplication, RED/USE/queue/fan-out coverage, and alert-driven scrutiny of code guarded by existing alerts/SLOs. Writes findings into the Oodle Change Context manifest and hands off fixes to oodle-monitors/oodle-drop-rules/oodle-log-metrics. Use when reviewing or authoring instrumentation changes."`, `tags: oodle,observability,code-review,instrumentation,cardinality,slo`, `globs: ""`.

Required sections (content from spec §Left):
1. **When to use / scope** — reviews the diff, not the whole repo; triggered in authoring/review of changes touching instrumented/service code.
2. **Backend discovery preamble** — discover observability MCP by tool shape (metrics/alerts/logs/traces); recognize the Oodle suite by shape for the tight path (env selection, `load_skill__*` when present); NEVER hard-code a server name; degrade gracefully (checks 1–3 run without a backend, check 4 softens).
3. **Check 1 — business context & cardinality placement** — low-card→metric labels (bounded: route/method/status/tier), high-card→span attrs/log fields (tenant_id/order_id/user_id); loop work carries an iteration identifier. Concrete good/bad examples.
4. **Check 2 — no duplicate telemetry** — each signal a distinct job; feeds manifest `~`/`-`.
5. **Check 3 — perf-coverage patterns** — RED (handlers), USE (pools), queue saturation (depth/lag), fan-out/N+1 (per-iteration I/O without a batched/measured span).
6. **Check 4 — alert-driven scrutiny** — resolve guarding alerts/SLOs from backend; demand debug-telemetry for guarded paths; a latency alert over an unlocalizable path is critical; record `guarded-by`; mark higher-risk.
7. **Output discipline** — report only confident findings, tagged critical/important/minor; write results into the manifest; hand off concrete fixes to the named existing skills. Explicitly: say what/why, not how-to-instrument.
8. **Reference** — link `oodle-o11y-context` for the manifest format.

- [ ] **Step 3: Structural validation**

Invoke `skill-quality-reviewer` on `skills/oodle-o11y-review/SKILL.md`. Fix critical/important findings inline.

- [ ] **Step 4: Behavioral dry-run**

For each `evals.md` scenario, reason through the skill as written and confirm it would drive the EXPECT outcome. Where a scenario fails, fix the SKILL.md wording. (Formal skill-creator Eval run happens in Task 5.)

- [ ] **Step 5: Commit**

```bash
git add skills/oodle-o11y-review/SKILL.md skills/oodle-o11y-review/evals.md
git commit -m "feat: add oodle-o11y-review skill (left-side telemetry review)"
```

---

## Task 3: Right — `oodle-triage`

**Files:**
- Create: `skills/oodle-triage/SKILL.md`
- Create: `skills/oodle-triage/evals.md`

- [ ] **Step 1: Write evals.md first**

Scenarios (min five):
1. **Single alert, Oodle present** — given a firing latency alert → EXPECT: ground truth in writing (scope + UTC/local window), confirm symptom with hard signals, read instrumentation from code, triangulate, correlate recent PR manifests, label findings Confirmed/Inferred/Unknown, ranked missing-evidence when unconfirmed.
2. **Single alert from a Linear ticket id** — EXPECT: fetch ticket for scope, produce report, offer to post back to the ticket.
3. **Manifest correlation** — a recent PR's `watch: N+1 on order lookup` exists → EXPECT: triage surfaces it as leading hypothesis and tries to falsify it.
4. **Auto-triage fan-out** — window with several firing + pending + muted + unrouted alerts → EXPECT: only firing considered, muted+unrouted suppressed (and stale tickets auto-closed), storm collapsed to one, fingerprint dedup, idempotent on rerun.
5. **Generic backend (no Oodle)** — EXPECT: discovery falls back to whatever observability MCP is present; Oodle-specific mechanics (ALERTS metric, muting_rules, has_notification) replaced by the generic equivalents or explicitly skipped.

Each with a one-line rubric.

- [ ] **Step 2: Write the SKILL.md**

Frontmatter: `name: oodle-triage`, `description: "Triages production alerts using observability signal. Mode A gathers confirmed-vs-inferred context for a single alert or ticket and updates the tracker; Mode B fans out over fired alerts in a window, suppresses muted/unrouted noise, dedupes via a fingerprint, and files/updates issues idempotently. Correlates alerts to recent changes via the Oodle Change Context manifest. Use when investigating an alert/incident or running scheduled oncall triage."`, `tags: oodle,observability,oncall,triage,incident,alerts`, `globs: ""`.

Required sections (content from spec §Right, distilled from oncall-debug.md + oncall-triage.md, MCP names generalized):
1. **When to use / modes** — Mode A (one alert/ticket) vs Mode B (window fan-out); how the input selects the mode.
2. **Backend + tracker discovery preamble** — same shape-based discovery as review; tracker MCP discovered too (Linear worked example, Jira analog); never hard-code names.
3. **Mode A — triage context** — the 7 steps: ground truth in writing (scope, UTC+local); confirm symptom with hard signals; read instrumentation from code (and note what's missing); triangulate metrics/logs/traces/profiles; **manifest correlation**; label Confirmed/Inferred/Unknown + falsify leading hypothesis; ranked missing-evidence; output report + offer to post to ticket.
4. **Mode B — auto oncall triage** — enumerate firing-only (never pending); enrich (still-active, noise category, muted, routed); suppress muted+unrouted (+ auto-close stale); collapse storms; per-alert investigation reuses Mode A depth-scaled; dedupe & file via fingerprint (create/comment/reopen); idempotent for `/loop`.
5. **Fingerprint block** — the stable dedup format (env + monitor + scope + links), matching oncall-triage's contract.
6. **Genericity vs Oodle-tight** — which parts are vendor-neutral discipline vs Oodle-path mechanics; graceful degradation.
7. **Reference** — link `oodle-o11y-context` (manifest read) and note handoffs to `oodle-metrics`/`oodle-logs`/`oodle-traces` for the Oodle CLI path.

- [ ] **Step 3: Structural validation**

Invoke `skill-quality-reviewer` on `skills/oodle-triage/SKILL.md`. Fix critical/important findings inline.

- [ ] **Step 4: Behavioral dry-run**

Reason each `evals.md` scenario through the SKILL as written; fix wording where it wouldn't drive the EXPECT. (Formal eval in Task 5.)

- [ ] **Step 5: Commit**

```bash
git add skills/oodle-triage/SKILL.md skills/oodle-triage/evals.md
git commit -m "feat: add oodle-triage skill (right-side context gathering + auto-triage)"
```

---

## Task 4: README + cross-links

**Files:**
- Modify: `README.md` (Available Skills table)

- [ ] **Step 1: Add three rows to the Available Skills table**

```markdown
| [oodle-o11y-context](skills/oodle-o11y-context/SKILL.md) | Change-context manifest — the PR block that carries a change's observability intent across the SDLC |
| [oodle-o11y-review](skills/oodle-o11y-review/SKILL.md) | Left-side telemetry review — business context, cardinality, duplication, RED/USE/fan-out, alert-driven scrutiny |
| [oodle-triage](skills/oodle-triage/SKILL.md) | Right-side triage — single-alert context gathering and windowed auto-triage with fingerprint dedup |
```

- [ ] **Step 2: Commit**

```bash
git add README.md
git commit -m "docs: list o11y-in-SDLC skills in README"
```

---

## Task 5: skill-creator Eval + Improve pass

**Files:** (edits to the three SKILL.md as eval findings require)

- [ ] **Step 1: Run skill-creator Eval mode**

Run `/skill-creator` → Eval mode against `skills/oodle-o11y-review/evals.md` and `skills/oodle-triage/evals.md` (Executor runs the skill per prompt, Grader scores vs each rubric).
Expected: each scenario passes its rubric. Record any failures.

- [ ] **Step 2: Improve mode on failures**

For any failing scenario, use skill-creator Improve mode (Analyzer suggests SKILL.md edits); apply the minimal wording change and re-run that scenario.

- [ ] **Step 3: Final quality review across the suite**

Invoke `skill-quality-reviewer` on all three SKILL.md files together; fix remaining critical/important findings.

- [ ] **Step 4: Commit**

```bash
git add skills/oodle-o11y-context/SKILL.md skills/oodle-o11y-review/SKILL.md skills/oodle-triage/SKILL.md skills/oodle-o11y-review/evals.md skills/oodle-triage/evals.md
git commit -m "test: harden o11y skills against skill-creator evals"
```

---

## Self-Review (completed by plan author)

**Spec coverage:** spine (Task 1), left four checks + discovery + gating + handoffs (Task 2), right Mode A/B + fingerprint + genericity (Task 3), README/manifests note (Task 4, manifests confirmed no-op), non-goals enforced via section outlines (no templates/references/hooks), never-hard-code-MCP stated in both discovery preambles. Skill-creator folded in (Task 5). Covered.

**Placeholder scan:** each task names exact files, exact frontmatter, exact section content, exact commit commands. Eval scenarios are concrete. No TBDs.

**Type consistency:** skill slugs (`oodle-o11y-context`, `oodle-o11y-review`, `oodle-triage`), the `## Oodle Change Context` header, and field keys (`type/touches/telemetry/guarded-by/watch/gaps`) are used identically across all tasks and match the spec.
