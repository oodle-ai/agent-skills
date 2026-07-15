# oodle-triage — eval scenarios

Behavioral fixtures for skill-creator Eval mode. Each scenario gives a PROMPT,
EXPECT, and a one-line RUBRIC.

---

## S1 — Single firing alert (Oodle present)

**PROMPT:** "Triage the firing `HighCheckoutLatency` alert on checkout-svc in
us1." An Oodle-shaped observability MCP is available.

**EXPECT:** Mode A. Write ground truth first (service/env/pod scope + window in
UTC *and* reporter-local). Confirm the symptom with hard signals before
theorizing. Read the service code to get the *actual* metric/span/log names and
note what is not captured. Triangulate metrics/logs/traces. Label every finding
`Confirmed | Inferred | Unknown`; try to falsify the leading hypothesis. If
unconfirmed, output a ranked missing-evidence list. Never hard-code the MCP
server name — discover it by shape.

**RUBRIC:** Pass if it writes scope-with-dual-timezone, confirms symptom from
signals, and labels findings Confirmed/Inferred/Unknown with a missing-evidence
list when unconfirmed.

---

## S2 — Alert from a Linear ticket id

**PROMPT:** "Triage D-7442." (A Linear ticket id.)

**EXPECT:** Mode A. Fetch the ticket via the tracker MCP for scope; run the Mode
A loop; produce the disciplined report; offer to post it back as a comment on the
ticket.

**RUBRIC:** Pass if it pulls scope from the ticket and offers to write the report
back to the tracker.

---

## S3 — Manifest correlation

**PROMPT:** "Triage a latency regression on checkout-svc." A recently merged PR
touching checkout-svc has an O11y Change Context block with
`watch: N+1 on order lookup under load`.

**EXPECT:** Mode A step 5 finds the recent PR touching the alerting service,
reads its manifest, and raises `N+1 on order lookup` as the **leading
hypothesis** — then tries to falsify it with a measurement rather than asserting
it. `guarded-by` in the manifest is used to confirm the alert↔code link.

**RUBRIC:** Pass if the manifest `watch` item becomes the leading hypothesis and
the skill attempts to falsify it.

---

## S4 — Auto-triage fan-out with noise

**PROMPT:** "Run oncall triage for the last 6h; file under Linear parent OBS-1234."
The window contains: two firing alerts, one pending-only condition, one firing
but muted alert, one firing but unrouted (`has_notification: false`) monitor, and
a storm of 30 series from one monitor.

**EXPECT:** Mode B. Consider **firing only** (drop the pending condition). Enrich
each with still-active / noise-category / muted / routed. **Suppress** the muted
and unrouted alerts — do not file them, and auto-close any stale ticket for them.
Collapse the 30-series storm into **one** logical alert/ticket. Dedupe against
existing OBS-1234 sub-issues via the fingerprint block; create/comment/reopen as
appropriate. Rerunning the same window changes nothing (idempotent).

**RUBRIC:** Pass if pending is excluded, muted+unrouted suppressed, storm
collapsed to one, and dedup is fingerprint-based and idempotent.

---

## S5 — Generic backend (no Oodle)

**PROMPT:** "Triage the firing latency alert." Only a non-Oodle observability MCP
(e.g. a generic Prometheus/Grafana-shaped MCP) is present.

**EXPECT:** Discovery binds to the generic MCP by shape. The vendor-neutral
discipline runs fully (ground truth, confirm symptom, triangulate, label
findings). Oodle-specific mechanics (the `ALERTS` metric shape, `muting_rules`
matcher semantics, `has_notification` routing) are replaced by the backend's
equivalents or explicitly skipped where the concept is absent — never invented.

**RUBRIC:** Pass if it runs the neutral discipline on the generic MCP and does not
assume Oodle-only mechanics exist.
