# Canonical test-plan format

Shared with the `test-planner` CLI — plans written by either tool must parse
identically, so keep this format byte-compatible. One file per ticket:
`plans/<TICKET-KEY>.md`.

```markdown
# Test Plan — <TICKET-KEY>: <ticket summary>

Generated: <ISO-8601 timestamp>

## AC issues            <!-- section present only when triage found problems -->

- **AC3** — not testable as written ("should be fast"); propose: "p95 GET
  /orders latency < 300ms at 100 rps" — needs product confirmation.

## Acceptance criteria

- **AC1** — <the criterion, verbatim or minimally normalized>
- **AC2** — ...

## Planned tests

### T1: <imperative test title>

- Type: `happy` | `negative` | `edge` | `failure`
- Layer: `unit` | `integration` | `contract` | `smoke`
- Verifies: AC1            <!-- comma-separated AC ids, or `implied` -->

```gherkin
Given <precondition>
When <action>
Then <observable outcome>
And <further assertions>
```
```

Rules:

- AC ids are `AC<n>`, numbered in ticket order; test ids are `T<n>`,
  contiguous from T1.
- `Verifies:` must reference existing AC ids. `implied` is reserved for
  failure-path tests the plan added beyond the ticket's ACs.
- Every AC appears in at least one test's `Verifies` — a plan violating
  this is invalid; fix it before emitting.
- Gherkin blocks only for `integration` (and `smoke`) layers. `unit`-layer
  tests replace the Gherkin block with a one-line `Case:` description —
  unit cases belong in Go table tests, not feature files.
- Titles are behavior statements ("Pay a non-pending order is rejected"),
  never implementation statements ("orderService.Pay returns error").
