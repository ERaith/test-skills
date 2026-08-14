# Canonical test-plan format

The plan is a **generalized coverage outline**: per acceptance criterion, the
assertions that MUST be covered — *what* to check, not *how*. The developer
writes the actual gobdd scenarios (against the `@step` catalog, per
gobdd-standards.md); this plan says what those tests must prove, and the
reviewer checks each required assertion against the MR's tests. **The plan
does NOT contain Gherkin or feature files.** One file per ticket:
`plans/<TICKET-KEY>.md`.

```markdown
# Test Plan — <TICKET-KEY>: <ticket summary>

Generated: <ISO-8601 timestamp>

## AC issues            <!-- only when triage found problems -->

- **AC3** — not testable as written ("should feel responsive"); no observable,
  no threshold. Propose: "p95 POST /confirm latency < 300ms at 100 rps", once
  product sets an SLO. No coverage planned against it — flagged, not planned
  around.

## Coverage outline

One block per acceptance criterion. Each required assertion names the
**observable to check** (status, error code, stored row, published event,
field value) — concrete enough for the reviewer to verify a test proves it,
general enough that the dev picks the exact gobdd steps.

### AC1 — <criterion, verbatim or minimally normalized>

- Layer: `acceptance` (gobdd) | `unit` | `contract`
- Required assertions:
  - **AC1.1** [happy] confirming a pending reservation → response 200 and
    status "confirmed"
  - **AC1.2** [state] the stored row (not only the response) shows "confirmed"
- PIBs: row, us            <!-- only when the ticket spans regions -->

### AC3 — <criterion>

- Layer: `acceptance`
- Required assertions:
  - **AC3.1** [negative] confirm on an already-confirmed or cancelled
    reservation → 409, error code "invalid_transition"
  - **AC3.2** [state] status unchanged after the rejected transition

## Implied coverage            <!-- required but stated by no AC -->

- **IMPL.1** [failure] unknown id on confirm/cancel → 404
  "reservation_not_found"
- **IMPL.2** [dual-write] the action persists AND publishes → assert BOTH the
  SQL row and the Kafka event; and the failure half (write fails → no event)
```

Rules:

- AC ids are `AC<n>`, numbered in ticket order. Required-assertion ids are
  `AC<n>.<m>` under their AC; implied ones are `IMPL.<m>`.
- **Every testable AC has ≥1 required assertion.** An untestable AC goes in
  **AC issues** with NO assertions — flagged for a human, never planned around
  with vague checks. A plan that invents coverage for an unmeasurable AC is
  wrong.
- Each assertion states an **observable outcome** and, where the AC names one,
  the exact value (status code, error code, field). "[state]" assertions pin
  the stored row / published event, not just the API response.
- Type tags: `[happy]` `[negative]` `[edge]` `[failure]` `[state]`
  `[dual-write]`. A perf/SLO concern is flagged in AC issues, not asserted.
- Layer: `acceptance` (gobdd) for behavior crossing a boundary (HTTP, Kafka,
  DB); `unit` for pure logic/mapping; `contract` for shared schema/events.
- **No Gherkin, no feature files, no step phrasing.** If a required assertion
  would need a step that isn't in the `@step` catalog, note
  "(may need a new gobdd step)" — do not invent the phrasing.

> Divergence from the `test-planner` CLI: the CLI emits `## Planned tests`
> with Gherkin blocks; this outline format replaces that with per-AC required
> assertions and no Gherkin. The two are no longer byte-compatible — the CLI
> needs a matching update, or its Gherkin blocks are treated as advisory. The
> shared vocabulary (`covered`/`partial`/`missing`, confidence buckets) is
> unchanged.
