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

**Dispatch the evidence wave** — three parallel read-only agents, one
message, prompt templates and merge rules in
[agent-fanout.md](${CLAUDE_PLUGIN_ROOT}/skills/test-planner/references/agent-fanout.md).
Use those templates verbatim; improvised scopes are why two reviews of one
MR disagree. Step 1's inputs are *not* fanned out — the diff, ticket and
plan are your anchor and you read them yourself.

- **B1 — unit tests:** `*_test.go` in and around the changed packages —
  every test function, `t.Run` subtest and table-case name, and, from the
  body, **what each actually asserts**, quoted. Not what the name claims.
  Skips and build guards reported with them.
- **B2 — feature files:** `features/**/*.feature` — every scenario, its
  tags (`@integration`, `@smoke`, `@wip` — anything `@wip` is not
  evidence), and every step **verbatim**.
- **B3 — step wiring:** every step registration (`ctx.Step(`,
  `InitializeScenario`) verbatim with the function it binds to, whether
  anything reaches that registration, and what each bound function actually
  touches — HTTP recorder, direct Spanner client, Kafka consumer, a fake,
  or nothing.

**B3 is blind to B2 on purpose, and you do the matching.** An agent that
reads a scenario and then hunts for its steps will accept a near-match,
because it knows what it wants to find; B3 is never told which scenarios
exist. Match every step string B2 returned against B3's pattern list
yourself. A scenario with one unmatched step, one `godog.ErrPending`
binding, or a registration nothing reaches is NOT evidence — it's a wish.
For Spanner/Kafka assertions, B3's "what it touches" line is the check: a
row-assertion step must reach the direct Spanner client, an event-assertion
step must consume the topic; one that goes back through the API's own read
path proves less than it appears to.

Then, in this session (never in a lane — execution is side-effecting and
its result is yours to witness): if the suite runs cheaply, run it.
`go test ./...` for unit; the `@integration`-tagged godog suite (via the
repo's integration build tag) when a container runtime is available. A
failing test is not evidence.

Re-find every lane citation by grep before it reaches the report — an
agent's citation is unverified until you have seen it, and a fan-out that
multiplies claims must not multiply unchecked claims. If a lane returned
`unavailable`, that part of the evidence surface was never inventoried:
state it once on the verdict line (Step 4) instead of quietly reviewing a
surface you did not see.

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
- **Double quotes mean verbatim-from-the-diff** (rubric evidence bar 2).
  Re-grep every `"..."` you write in an Evidence cell, *as written*,
  including the ones you are sure of. Elisions (`"...field is rejected"`),
  tidied-up plan sentences, and quoted labels you coined for a scenario all
  fail that grep. Command output, skeptic wording and your own summary are
  evidence too, but they are not citations: give them prose or backticks,
  never quotes.
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
  failure paths are gaps even when every AC is green. What each response
  must assert — including the two RFC 9110 `MUST`s (`WWW-Authenticate` on
  401, `Allow` on 405) — is in
  [test-design-techniques.md](${CLAUDE_PLUGIN_ROOT}/skills/test-planner/references/test-design-techniques.md)
  §7.
- **Edge-case gap check.** Re-derive the edge cases from the **ACs** using
  the same seven-step pipeline the planner used
  ([test-design-techniques.md](${CLAUDE_PLUGIN_ROOT}/skills/test-planner/references/test-design-techniques.md)),
  then diff that list against the Step 2 evidence. Derive from the ACs, never
  from the MR's tests — deriving from the code under review just re-describes
  what was built. Two outcomes, and the difference is not cosmetic:
  - the AC **itself names** the item (it states the 200-char limit, or names
    the 409 on an invalid transition) and nothing exercises it → the item is
    part of the AC's observable outcome: `partial`, axis named;
  - the item is **derived, not stated** (an untested invalid partition, an
    unmentioned boundary, a 405 path) → the AC's verdict is **unaffected**;
    it goes under **Gaps and recommended tests** with the technique named
    ("BVA, `title` length: no case at the storage limit").

  The second rule is what keeps the verdict column meaningful: the derived
  set is always larger than the AC set, so letting derived items downgrade
  ACs would make every review INSUFFICIENT and the column would stop
  discriminating. Gaps also never move confidence — that measures how sure
  you are of a verdict, not how much coverage exists.

## Step 3b — Try to break every `covered`

A `covered` verdict is a claim, and the claim is cheap until something has
attacked it. **Dispatch one skeptic agent per verdict you are about to call
`covered`, all in one message** (template in
[agent-fanout.md](${CLAUDE_PLUGIN_ROOT}/skills/test-planner/references/agent-fanout.md)).
Each is given one AC, one citation, the repo path and the attack list —
and nothing else: not the other verdicts, not your confidence, not the word
"covered". A skeptic shown a page of green verdicts calibrates to the page.
Skeptics run only on `covered` candidates; `partial` and `missing` are
already the conservative call.

With no subagent runtime, do the same attacks inline, one verdict at a
time, and state the limitation on the verdict line. The standard attacks:

- the assertion checks a proxy (status code, `err == nil`) instead of the
  outcome the AC actually names;
- the scenario is unwired or half-wired — one step has no registration;
- the fixture makes the assertion pass regardless of the behavior under
  test (a seeded row, a stubbed response asserted back);
- the row is asserted only through the API's own read path, so a symmetric
  write/read bug cancels out;
- the test is excluded in the configuration CI actually runs (build tag,
  tag expression, `t.Skip` behind an env var).

Record one outcome per verdict — the vocabulary is fixed, and it means the
same whether a skeptic agent or an inline pass produced it:

- **refuted** — the attack lands. It is not `covered`; downgrade to
  `partial` or `missing` and cite what broke it.
- **survived-caveat** — the attack does not land, but it surfaced something
  you could not fully rule out. Stays `covered`, confidence capped at
  `medium`, and the caveat goes in Notes.
- **survived** — the attack fails cleanly.

**The evidence bar is symmetric.** A refutation is a claim too: before a
`refuted` outcome moves a verdict, re-find the skeptic's citation. An
objection nobody can cite is not a refutation — record it as
`survived-caveat` with the doubt in Notes. A skeptic that returned nothing
usable is not a `survived` either; attack that verdict yourself before
keeping it. Downgrading on an uncitable objection would make the report
pessimistic in exactly the unfalsifiable way the rubric refuses to be
optimistic.

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

<!-- Every "..." above is verbatim from the diff; observed output and skeptic
     wording go in prose (`run output: expected 409, got 200`), not quotes. -->

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
