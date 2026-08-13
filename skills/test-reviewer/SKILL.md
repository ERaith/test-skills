---
name: test-reviewer
description: Review a merge request's test coverage against the ticket's test plan for Go Spanner+Kafka API services. Classifies each acceptance criterion as covered/partial/missing with cited evidence from Go unit tests and godog feature files. Advisory only — reports, does not gate. Input is an MR reference (URL, !id, or branch) and optionally the ticket key.
argument-hint: "[MR-REF] [TICKET-KEY]"
disable-model-invocation: true
---

# Test reviewer: MR + test plan → per-AC coverage verdict

You are the coverage reviewer. Your verdict is **traceability-based**: do
the MR's tests demonstrably cover the ticket's acceptance criteria, per the
test plan? Line-coverage percentages are at most a supporting signal —
never the verdict. Read
[references/completeness-rubric.md](references/completeness-rubric.md)
before classifying; it defines the verdicts, the evidence bar, and the
anti-gaming rules, and it wins whenever this file and intuition disagree.

Advisory mode: you produce a report (and optionally an MR comment). You do
not block anything. An SDET makes the merge call.

## Step 1 — Gather the three inputs

1. **The MR.** From URL/!id: `glab mr view $ARGUMENTS` and
   `glab mr diff` (add `--repo` when outside a checkout). From a branch:
   diff against the default branch. You need the full diff AND the repo
   tree — coverage evidence may live in test files the MR didn't touch.
2. **The ticket.** Key from the argument, else from the MR title/branch/
   description. Fetch the acceptance criteria via the Jira MCP tools
   (ToolSearch to find them); if unavailable, extract ACs from the MR
   description or ask. Never invent ACs.
3. **The plan.** `plans/<TICKET-KEY>.md` in the repo (canonical location;
   format in
   [plan-format.md](${CLAUDE_PLUGIN_ROOT}/skills/test-planner/references/plan-format.md)).
   If absent: offer to run the test-planner skill first, or — if the user
   wants the review now — derive a minimal ad-hoc plan from the ACs and
   label the report "reviewed against ad-hoc plan (no approved plan found)".

## Step 2 — Inventory the coverage evidence

Two evidence sources; inventory both — they satisfy different layers:

- **Go unit tests** — `*_test.go` in and around the changed packages.
  Enumerate test functions, `t.Run` subtests, and table-test case names.
  Read the bodies: note what each actually *asserts*, not what its name
  claims.
- **godog feature files** — `features/**/*.feature`. Enumerate scenarios
  and tags (`@integration`, `@smoke`, `@wip` — anything tagged `@wip` is
  not evidence). Then verify each relevant scenario is **wired**: grep the
  step registrations (`InitializeScenario`, `ctx.Step(`) for a pattern
  matching every step. A scenario with undefined or pending steps is NOT
  evidence — it's a wish. For Spanner/Kafka assertions, check the step
  implementation reaches the right place: a row-assertion step should use
  the direct Spanner client, an event-assertion step should consume the
  topic.
- If the suite runs cheaply, run it: `go test ./...` for unit; the
  `@integration`-tagged godog suite (via the repo's integration build tag)
  when a container runtime is available. A failing test is not evidence.

## Step 3 — Classify

For **each AC** and **each planned test**, assign `covered` / `partial` /
`missing` per the rubric. The mechanical rules:

- **Evidence must be cited and must exist.** Every non-`missing` verdict
  cites file + test/scenario name + the specific assertion satisfying the
  AC's observable outcome. Re-find every citation by grep before writing
  it — a citation you cannot re-find is a hallucination, and the verdict
  falls back to `missing`.
- **Layer must match the plan.** A planned `integration` test satisfied
  only by a unit test with mocked boundaries is `partial` — the assertion
  exists, but the behavior wasn't proven across the boundary the plan
  demanded. The reverse (integration evidence for a planned unit case) is
  acceptable; note it.
- **Assertion substance over test names** (rubric smells): a test named
  for the AC that never asserts the AC's outcome — calls the endpoint,
  checks only `err == nil` or status 200 — is `partial` at best.
- **Dual-write ACs need both halves.** Where the AC (or plan) demands a
  Spanner row AND a Kafka event, evidence asserting only one half is
  `partial`; say which half is missing.
- **Failure paths:** walk the rubric's checklists (HTTP, Kafka consumer,
  Spanner persistence) against every endpoint/consumer the diff touches,
  including the plan's `Verifies: implied` tests. Uncovered applicable
  failure paths are gaps even when every AC is green.

## Step 4 — Report

Emit the report in this shape (stable, so future tooling can parse it):

```markdown
# Coverage review — <TICKET-KEY> / MR <ref>

**Verdict: SUFFICIENT | INSUFFICIENT** (plan: plans/<KEY>.md | ad-hoc)

| AC | Verdict | Evidence |
|----|---------|----------|
| AC1 | covered | features/order_lifecycle.feature "Pay a pending order" — asserts status 200 + data.status "paid" |
| AC3 | partial | handlers/pay_test.go TestPay/rejects_paid — asserts 409 but not error.code |
| AC4 | missing | — |

## Planned-test verdicts
<same table for T1..Tn>

## Gaps and recommended tests
<per gap: what's missing, plus a ready-to-add skeleton — Gherkin reusing
the repo's existing step vocabulary for integration gaps, a Go table-test
case for unit gaps>

## Notes
<layer mismatches, unwired scenarios, name/assertion mismatches, dual-write
halves missing, line-coverage signal if computed>
```

`INSUFFICIENT` = any AC `missing`, or a judgment call per the rubric's
severity guidance (explain it when you make it). Offer — do not do
unprompted — to post the report as an MR comment (`glab mr note`). Keep
the report honest about its own limits: anything you could not verify
(suite didn't run, Jira unreachable, no container runtime), say so
explicitly rather than letting silence imply green.
