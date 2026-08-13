---
name: test-reviewer
description: Review a merge request's test coverage against the ticket's test plan for Go Spanner+Kafka API services. Classifies each acceptance criterion as covered/partial/missing with cited evidence from Go unit tests and godog feature files, plus a per-AC confidence (high/medium/low) and what would raise it. Advisory only — reports, does not gate. Input is an MR reference (URL, !id, or branch) and optionally the ticket key.
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

Tag every piece of evidence you inventory as **executed** (you ran it in this
review and it passed) or **static-only** (you read it). That tag is an input
to confidence in Step 3c, so record it as you go — reconstructing it at
report time is how "we ran the suite" quietly becomes true of tests that
never ran.

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

## Step 3b — Try to break every `covered`

A `covered` verdict is a claim, and the claim is cheap until something has
attacked it. For each verdict you are about to call `covered`, make one
honest attempt to refute it: assume the AC is *not* covered and go find the
reason. The standard attacks:

- the assertion checks a proxy (status code, `err == nil`) instead of the
  outcome the AC actually names;
- the scenario is unwired or half-wired — one step has no registration;
- the fixture makes the assertion pass regardless of the behavior under
  test (a seeded row, a stubbed response asserted back);
- the row is asserted only through the API's own read path, so a symmetric
  write/read bug cancels out;
- the test is excluded in the configuration CI actually runs (build tag,
  tag expression, `t.Skip` behind an env var).

Record one outcome per verdict:

- **refuted** — the attack lands. It is not `covered`; downgrade to
  `partial` or `missing` and cite what broke it.
- **survived-caveat** — the attack does not land, but it surfaced something
  you could not fully rule out. Stays `covered`, confidence capped at
  `medium`, and the caveat goes in Notes.
- **survived** — the attack fails cleanly.

Queue item 1 replaces this inline pass with one skeptic agent per `covered`
verdict, running in parallel. That vocabulary — refuted / survived-caveat /
survived — is the contract between the two; the mechanism changes, the
outcomes and their confidence consequences do not.

## Step 3c — Assign confidence

Give **every** verdict — AC and planned test alike — a `high` / `medium` /
`low` confidence, derived by the rubric's confidence table and caps from two
recorded facts: executed-vs-static-only (Step 2) and the Step 3b outcome.
Do not eyeball it, and do not let a clean-looking diff raise it.

Then, for every verdict below `high`, write the one-line **what would raise
it**: the specific action that moves this verdict up a bucket. Finally take
the overall report confidence — the lowest per-AC confidence, capped for an
ad-hoc plan, an unrunnable suite, or ACs that never came from the ticket
system.

## Step 4 — Report

Emit the report in this shape (stable, so future tooling can parse it):

```markdown
# Coverage review — <TICKET-KEY> / MR <ref>

**Verdict: SUFFICIENT | INSUFFICIENT** (plan: plans/<KEY>.md | ad-hoc)
**Confidence: medium** — static-only: the @integration suite did not run (no
container runtime)

| AC | Verdict | Confidence | Evidence |
|----|---------|------------|----------|
| AC1 | covered | high | features/order_lifecycle.feature "Pay a pending order" — asserts status 200 + data.status "paid"; executed, survived refutation |
| AC2 | covered | medium | features/order_lifecycle.feature "Paying publishes order.paid" — asserts the event; static-only |
| AC3 | partial | high | handlers/pay_test.go TestPay/rejects_paid — asserts 409 but not error.code |
| AC4 | missing | high | — |

**What would raise it** (one line per verdict below `high`)
- AC2 (medium → high): run the `@integration` suite (`go test -tags
  integration ./features/...` with a container runtime) — the evidence is
  read-only so far.

## Planned-test verdicts
<same four-column table for T1..Tn, same "what would raise it" treatment>

## Gaps and recommended tests
<per gap: what's missing, plus a ready-to-add skeleton — Gherkin reusing
the repo's existing step vocabulary for integration gaps, a Go table-test
case for unit gaps>

## Notes
<layer mismatches, unwired scenarios, name/assertion mismatches, dual-write
halves missing, refutation caveats from Step 3b, line-coverage signal if
computed>

Confidence: <n> high / <n> medium / <n> low across <n> verdicts.
```

`INSUFFICIENT` = any AC `missing`, or a judgment call per the rubric's
severity guidance (explain it when you make it). Confidence is not a
tiebreaker: a `low`-confidence `covered` is still `covered`, and a
`high`-confidence `partial` is still a gap. Low confidence means "look here
yourself", which is why it goes on the verdict line rather than quietly
softening the verdict. Offer — do not do
unprompted — to post the report as an MR comment (`glab mr note`). Keep
the report honest about its own limits: anything you could not verify
(suite didn't run, Jira unreachable, no container runtime), say so
explicitly rather than letting silence imply green.
