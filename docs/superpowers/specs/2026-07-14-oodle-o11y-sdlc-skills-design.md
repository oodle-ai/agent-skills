# Oodle Observability-in-SDLC Skills — Design

**Date:** 2026-07-14
**Status:** Approved design, pending implementation plan
**Repo:** `oodle-ai/agent-skills`

## Problem & thesis

Observability tooling for AI coding agents today (e.g. the nexus-labs
`backend-observability` / `web-observability` / `agent-observability` plugins,
Dash0, Elastic agent-skills) over-invests in *how to instrument* — per-language
code templates, SDK setup, reference trees. Modern coding agents already hold
that knowledge in their training set. The unmet need is the **loop/judgment
layer**: does emitted telemetry carry the right business context at the right
cardinality, is it non-duplicative, is coverage aligned to what production
actually alerts on, and does context flow from authoring through review into
production triage.

This suite is **tightly scoped to that judgment + connective tissue**, covering
both the **left** (authoring/review) and **right** (production monitoring,
triage/releases) of the SDLC. It is **vendor-generic** (binds to whatever
observability + issue-tracker MCP is present) and **tighter when Oodle is
present** (recognizes the Oodle tool suite by shape).

## Non-goals (the sharp edge)

- ❌ No per-language instrumentation how-to, code templates, or SDK setup guides.
- ❌ No `references/` or `templates/` trees. Each skill is one lean `SKILL.md`.
- ❌ No hooks / slash-commands / subagents — pure portable skills, consistent
  with the existing 12 skills (run on Claude Code, Cursor, Gemini, Codex,
  Windsurf via `npx skills add` / `oodle skills install`).
- ❌ No duplication of the existing 12 skills. When a finding implies an action
  (add a monitor, drop a cardinality bomb, derive a log metric), the skill
  **hands off** to `oodle-monitors` / `oodle-drop-rules` / `oodle-log-metrics`
  rather than re-teaching them.
- ❌ **Never hard-code an Oodle MCP server name.** Server names vary
  (`oodle-ai-us1`, `ap1`, `staging`, `dev`, customer-custom). The backend is
  discovered at runtime by tool *shape*, never by literal prefix.

## Suite shape — three portable skills, one loop

| Skill | Side | Job |
|---|---|---|
| `oodle-o11y-context` | spine | The change-context manifest convention — the `## Oodle Change Context` PR block: fields, how an authoring agent fills it, how downstream agents read it. Small; a shared contract the other two reference. |
| `oodle-o11y-review` | left | During authoring/review: judge telemetry for business-context attributes, cardinality placement, duplication, RED/USE/queue/fan-out coverage, and alert-driven scrutiny. Emits findings into the manifest. |
| `oodle-triage` | right | Given an alert/ticket → gather confirmed-vs-inferred production context (Mode A); fan-out/dedupe/file over fired alerts (Mode B). Reads the manifest to correlate recent risky changes. |

## Spine — `oodle-o11y-context`

A single markdown block with a stable header, locatable by shape (as
`oncall-triage` locates its `Fingerprint` block):

```markdown
## Oodle Change Context
type: perf, bugfix                 # bugfix | perf | feature | critical-path | refactor | infra
touches: checkout-svc, order-repo  # deploy/service units changed
telemetry:
  + span checkout.place_order {attr: tenant, order_id}   — trace slow orders per tenant
  ~ metric checkout_latency_seconds (histogram)          — widened buckets; guards SLO
  - metric checkout_legacy_timer                          — removed; duplicate of above
guarded-by: SLO checkout-p99, monitor HighCheckoutLatency  # existing alerts over touched code
watch: N+1 on order lookup under load; cache hit-rate regression
gaps: no per-item span in the loop — deferred (cardinality); triage can't attribute per-SKU
```

**Field logic** (each ties to a stated requirement):

- `type` — sets scrutiny level for both reviewer and triage.
- `telemetry` with `+ / ~ / -` and a *purpose* per line — enforces "every signal
  serves a purpose, no duplicates."
- `guarded-by` — the alert-driven link; resolved from the live backend during review.
- `watch` — the author/reviewer's risk hypotheses; triage reads this first.
- `gaps` — telemetry deliberately not added; tells triage what the data cannot
  answer (mirrors `oncall-debug`'s "know what's missing" discipline).

**Lifecycle (the loop):**

1. Authoring agent drafts `type / touches / telemetry / watch` when opening the PR.
2. `oodle-o11y-review` enriches: resolves `guarded-by` from the live backend,
   appends `gaps`, may upgrade `type` → `critical-path` if touched code sits
   under a critical SLO.
3. Merge → the block lives in PR history.
4. On an alert, `oodle-triage` finds recent PRs touching the alerting service
   and reads their manifests — `watch`+`type` focus the hunt, `guarded-by`
   confirms the alert↔code link, `gaps` explains missing evidence.

**Carrier:** PR description only (read back at triage via the GitHub/Linear MCP).
No git trailer or tracked repo file in v1 — keep it simple.

**Relevance gate:** the block is only expected when a change touches
instrumented/service code or is `perf` / `critical-path` / `bugfix`. Docs/trivial
changes skip it — `oodle-o11y-review` decides relevance.

The `oodle-o11y-context` SKILL is small: it defines this format + relevance gate
and is referenced by the other two skills rather than invoked much on its own.

## Left — `oodle-o11y-review`

**Trigger:** authoring/review of a change touching instrumented/service code, or
on demand. Reviews the **diff**, not the whole repo.

**Preamble — backend discovery:** detect the observability MCP by tool shape
(metrics/alerts/logs/traces capabilities); if the Oodle tool suite is
recognized, use the tight path (env selection, `load_skill__*` where relevant) —
never a hard-coded server name. With no backend MCP, checks 1–3 still run; only
check 4 (alert-driven) degrades gracefully.

**The four checks:**

1. **Business context & cardinality placement.** Where request context carries
   tenant/account/user, that identity must ride the telemetry, placed correctly:
   **low-cardinality on metric labels** (bounded: route, method, status, tier),
   **high-cardinality on span attributes / log fields** (tenant_id, order_id,
   user_id). Work in a loop must carry an iteration identifier so it is
   attributable. Flag: high-card label on a metric (→ `oodle-drop-rules`), or a
   missing tenant attribute on a span in tenant-scoped code.
2. **No duplicate telemetry.** Every added metric/span/log serves a distinct
   job; flag a new signal overlapping an existing one (e.g. a timer duplicating
   a histogram). Feeds the manifest's `~`/`-` lines.
3. **Perf-coverage patterns.** On touched code: **RED** on request handlers,
   **USE** on resource pools, **queue saturation** (depth / consumer-lag), and
   **fan-out / N+1** (a loop issuing per-iteration I/O with no batched or
   measured span). Report missing coverage as a gap.
4. **Alert-driven scrutiny (the differentiator).** Resolve which existing
   alerts/SLOs guard the touched code from the backend. For guarded code, demand
   the telemetry needed to *debug that alert* exists — a latency alert guarding a
   path that lacks the spans/attrs to localize slowness is a **critical**
   finding. Mark the change higher-risk; record `guarded-by` in the manifest.

**Output discipline:** report only findings confident to be real, tagged
`critical / important / minor` (avoid-alert-fatigue ethos, not a nitpick wall).
Write results into the manifest (`guarded-by`, `gaps`, telemetry purposes). Hand
off concrete fixes to `oodle-monitors` / `oodle-drop-rules` /
`oodle-log-metrics`. It says *what* and *why*; the agent implements *how*.

## Right — `oodle-triage`

Same backend-discovery preamble. Tracker writes go through whatever issue-tracker
MCP exists — **Linear as the worked example, Jira as the analog** — using a
stable `Fingerprint` block for dedup.

**Mode A — Triage context (one alert / ticket)** — distilled from `oncall-debug`:

1. Ground truth in writing — scope (service/env/pod, exact window in UTC *and*
   local), from the ticket or description.
2. Confirm the symptom with hard signals before theorizing.
3. Read instrumentation from code to get the *actual* metric/log/span names — and
   note what is *not* captured.
4. Triangulate metrics/logs/traces/profiles, grouping by attributing dimensions.
5. **Manifest correlation (loop closes here):** find recent PRs touching the
   alerting service, read their `## Oodle Change Context` — `watch`+`type` focus
   hypotheses, `guarded-by` confirms the alert↔code link, `gaps` explains missing
   evidence.
6. Label every finding `Confirmed | Inferred | Unknown`, try to *falsify* the
   leading hypothesis, and when you cannot confirm, emit a ranked
   missing-evidence list instead of a tidy story.
7. Output the disciplined report; offer to post it to the ticket.

**Mode B — Auto oncall triage (fan-out over fired alerts)** — distilled from
`oncall-triage`:

1. Enumerate **firing** (never pending) alerts in a window, across environments.
2. Enrich: still-active, noise category, **muted**, **routed**.
3. **Suppress** muted + unrouted (do not file; auto-close stale tickets for them).
4. Collapse storms → one logical alert.
5. Per-alert investigation reuses Mode A, depth-scaled by severity; manifest
   correlation feeds probable-cause / suggested-fix.
6. Dedupe & file against the tracker via the `Fingerprint` block —
   create / comment / reopen; **idempotent** for a `/loop` cadence.

**Genericity vs Oodle-tight:** the discipline (confirmed-vs-inferred,
firing-only, suppress muted/unrouted, fingerprint dedup) is vendor-neutral.
Oodle-specific mechanics (`ALERTS` metric shape, `muting_rules` matcher
semantics, `has_notification` routing) live in the Oodle path; a generic backend
uses its equivalents, degrading gracefully where a concept is absent.

## Cross-cutting

- **Backend discovery contract** stated inline in both active skills (self-contained
  skills): discover observability MCP by capability/shape → select environment →
  recognize the Oodle suite by shape for the tight path → never hard-code a
  server name → degrade gracefully when absent.
- **Tracker discovery:** same pattern for the issue-tracker MCP (Linear worked
  example, Jira analog).
- **Output gating everywhere:** report only findings confident to be real,
  tagged `critical / important / minor`.
- **Portability & install:** pure `SKILL.md`, no Claude-Code-only constructs;
  ships through `npx skills add oodle-ai/agent-skills` / `oodle skills install`.
- **Manifests to update:** README skill table, `.claude-plugin/plugin.json`,
  `.cursor-plugin/plugin.json`, `gemini-extension.json` (as needed to register
  the three new skills).
- **Handoffs, no duplication:** review → `oodle-monitors` / `oodle-drop-rules` /
  `oodle-log-metrics`; triage leans on `oodle-metrics` / `oodle-logs` /
  `oodle-traces` for the Oodle CLI path where useful.

## Deliverables

1. `skills/oodle-o11y-context/SKILL.md`
2. `skills/oodle-o11y-review/SKILL.md`
3. `skills/oodle-triage/SKILL.md`
4. README table + plugin/extension manifest updates registering the three skills.

## Open questions / deferred

- Git commit trailer or tracked-file carrier for the manifest (deferred; PR-only in v1).
- Jira-specific field mapping detail (Linear is the worked example; Jira analog
  described but not exhaustively specified).
- Whether `oodle-triage` Mode B should ship a companion `/loop` usage note or
  leave cadence to the operator (lean: document, don't automate).
