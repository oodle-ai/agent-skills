# oodle-o11y-review — eval scenarios

Behavioral fixtures for skill-creator Eval mode. Each scenario gives a PROMPT
(the situation the agent faces) and EXPECT (what a passing run of the skill
produces), plus a one-line RUBRIC the Grader scores against.

---

## S1 — High-cardinality metric label

**PROMPT:** Review this diff:
```go
ordersTotal := prometheus.NewCounterVec(
    prometheus.CounterOpts{Name: "orders_total"},
    []string{"order_id", "status"},   // new
)
ordersTotal.WithLabelValues(order.ID, "created").Inc()
```

**EXPECT:** `order_id` flagged as a **critical** high-cardinality label on a
metric; recommend keeping bounded labels (`status`) and moving `order_id` to a
span attribute or log field; hand off to `oodle-drop-rules` for mitigation if the
metric already ships. Manifest `telemetry` line records the counter with its
purpose.

**RUBRIC:** Pass if it flags `order_id` as high-cardinality on a metric AND
proposes relocation to span/log (not a metric label).

---

## S2 — Missing tenant context on a span

**PROMPT:** Review this diff (handler runs inside tenant-scoped middleware):
```python
with tracer.start_as_current_span("billing.charge") as span:
    span.set_attribute("amount", amount)
    charge(account, amount)
```

**EXPECT:** Flag that tenant/account identity from request context is not on the
span; recommend adding `tenant_id`/`account_id` as **span attributes**
(high-cardinality is acceptable on spans, not on metric labels). Note this in the
manifest `telemetry` purpose.

**RUBRIC:** Pass if it recommends tenant/account as a span attribute and states
high-card is fine on spans but not metric labels.

---

## S3 — Duplicate telemetry

**PROMPT:** Review this diff — the service already emits
`http_request_duration_seconds` (histogram). The diff adds:
```go
requestTimer := metrics.NewTimer("checkout_request_ms")
```

**EXPECT:** Flag the new timer as **duplicate** of the existing histogram; each
signal must serve a distinct job; recommend removing it (manifest `-` line with
reason) or justifying a distinct purpose.

**RUBRIC:** Pass if it identifies the overlap with the existing histogram and
recommends removal or explicit distinct justification.

---

## S4 — Fan-out / N+1 with no measured span

**PROMPT:** Review this diff:
```python
for item in order.items:              # new loop
    price = pricing_client.get(item.sku)   # per-item remote call
    total += price
```

**EXPECT:** Flag as an **N+1 / fan-out** perf-coverage gap; recommend either a
batched call or a span measuring the loop (with an iteration/count attribute so
it is attributable), and record the risk in the manifest `watch` line.

**RUBRIC:** Pass if it identifies per-iteration I/O as N+1 and asks for batching
or a measured span with an identifier.

---

## S5 — Alert-driven scrutiny (and no-backend degradation)

**PROMPT:** Review a diff to `checkout-svc`'s order-placement path. An existing
Oodle SLO `checkout-p99` and monitor `HighCheckoutLatency` guard this path, but
the changed code adds no spans that localize where latency is spent. (Run once
with an observability MCP present; run once with none present.)

**EXPECT (backend present):** Resolve the guarding alerts/SLOs from the backend
by tool shape (never a hard-coded server name); raise a **critical** finding that
the guarded path lacks the telemetry needed to debug that alert; record
`guarded-by: SLO checkout-p99, monitor HighCheckoutLatency` in the manifest and
mark the change higher-risk; recommend the localizing spans.

**EXPECT (no backend):** Checks 1–3 still run; check 4 is explicitly reported as
degraded ("no observability backend detected — cannot resolve guarding alerts")
rather than silently skipped.

**RUBRIC:** Pass if (a) with backend it records `guarded-by` and raises a critical
telemetry-gap finding, and (b) without backend it still runs structural checks and
names the degradation explicitly.
