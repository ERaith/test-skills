---
name: test-planner
description: Generate a per-acceptance-criterion test plan from a Jira ticket, ready for godog/Go implementation against Spanner+Kafka services. Use right after ticket breakdown, before development starts. Input is a Jira ticket key (e.g. PROJ-123), a Jira URL, or pasted ticket content.
argument-hint: "[TICKET-KEY]"
disable-model-invocation: true
---

# Test planner: Jira ticket → per-AC test plan

You are producing the test plan for one ticket. The plan is the contract the
test-reviewer skill later verifies the MR against, so precision here is what
makes the whole workflow work. The output format is canonical and shared with
the team's `test-planner` CLI — read
[references/plan-format.md](references/plan-format.md) before writing the plan.

## Step 1 — Resolve the argument, then fan out for context

Resolve the argument first:

- **Ticket key or URL** → the ticket, its epic and its links are gathered
  by lane **A1**; the linked Confluence spec by lane **A2** (see below).
- **Pasted content** → use as-is (no A1), but say which context you're
  missing (epic, spec) and ask for it only if the ACs don't stand alone.
- **No Jira access and nothing pasted** → stop and ask for the ticket
  content. Never invent acceptance criteria.

Then locate the target repo (from the ticket, or ask once) — prefer a local
checkout; otherwise pull context via `glab` (`glab repo clone`, or
`glab api` for file reads on GitLab).

**Dispatch the context wave.** Three parallel read-only agents, prompt
templates and merge rules in
[references/agent-fanout.md](references/agent-fanout.md) — use those
templates rather than improvising, because the plan's reproducibility
depends on the lanes having fixed scope:

- **A1 — ticket graph:** the ticket verbatim, the epic, and every linked /
  blocking / dependent ticket, including any AC in a *linked* ticket that
  constrains this one. Constraints inherited from a blocker are the ones
  plans usually miss.
- **A2 — spec:** the linked Confluence page(s) — specifically what the spec
  says that the ticket does not.
- **A3 — repo suite inventory:** `features/` scenarios and tags, every
  godog step registration verbatim, how the integration suite is executed,
  and the unit-test layout.

A1+A3 go out in one message; A2 joins them if a spec URL is already known,
otherwise it goes the moment A1 returns with the URLs. Without a subagent
runtime, do all three inline in lane order and say so in the summary — never
skip a lane.

## Step 2 — Merge the wave

Agents gather; you decide. Nothing from a lane reaches the plan unchecked:
re-find any citation you intend to quote, and if a lane came back
`unavailable`, either do its work inline or name the gap in the summary —
never both skip it and stay quiet.

**Number the ACs — once, here, from A1's verbatim AC block** (or the pasted
text): `AC1..ACn` in document order. No lane numbers ACs and no lane invents
one; A2's constraints and A3's vocabulary never become ACs. A spec
constraint the ticket omits becomes a `Verifies: implied` test in Step 4 and
a line under **AC issues** — not an AC the planner minted for itself. If the
ticket has no explicit AC section, extract testable statements from the
description and present them as "derived ACs — confirm before relying on
this plan."

Two things A3 buys: **layer assignment** grounded in what the repo actually
has (unit vs godog integration vs contract), and **step-vocabulary reuse** —
draft Gherkin must use the step patterns A3 returned wherever they fit, not
invent near-duplicates (see
[references/godog-standards.md](references/godog-standards.md)). If A3 came
back `unavailable` (no checkout reachable), the layer assignments are
ungrounded guesses — say that in the summary rather than presenting them as
fitted to the repo.

## Step 3 — Testability triage (do this BEFORE planning)

Judge each AC against: observable outcome stated? inputs/preconditions
stated? unambiguous pass/fail? If an AC fails the bar (e.g. "search should
be fast", "handles errors gracefully"), do not write vague tests around it —
flag it at the top of the plan under **AC issues**, with a concrete rewrite
suggestion ("propose: p95 search latency < 300ms at 100 rps" style). If most
ACs are untestable, stop and report back instead of emitting a vacuous plan:
pushing ambiguity back to the ticket before code exists is this skill's
highest-value output.

**Triage the spec too, not just the ACs.** Design notes and Confluence pages
carry rules the ticket never mentions, and some of them are openly unsettled —
"not decided yet", "TBD", "we should probably", a question left in a comment.
Those are in scope for the plan and they get the same treatment as a vague AC:
list them under **AC issues** (labelled as a spec open question, with the
source), and plan no test that depends on the answer. There are two ways to
get this wrong and both showed up in practice:

- *dropping it* — the rule never appears in the plan, so nobody learns the
  ticket shipped with an open question underneath it;
- *deciding it* — writing a test that asserts whatever the code does today,
  which silently promotes an undecided question into a regression lock and
  converts the next product decision into a test failure.

Asserting current behaviour is only correct once someone has decided that is
the behaviour. Until then, name it and leave it uncovered on the record.

## Step 4 — Write the plan

One section per planned test, following plan-format.md exactly:

- Every test declares `Type` (`happy` / `negative` / `edge` / `failure`),
  `Layer` (`unit` / `integration` / `contract` / `smoke`), and `Verifies`
  (the AC ids it covers).
- **Every AC must be verified by at least one test.** Check this before
  emitting; it is the invariant the reviewer enforces later.
- Integration-layer tests get draft Gherkin in godog style (godog-standards
  has the authoring rules — declarative, behavior-per-scenario, reuse steps).
  Unit-layer tests get a one-line description of the case, not Gherkin.
- **Derive the edge cases per AC — don't wait for inspiration.** Run the
  seven-step pipeline in
  [references/test-design-techniques.md](references/test-design-techniques.md)
  over each AC in order: variables/outcome → equivalence partitions
  (including the invalid ones) → boundaries on ordered partitions →
  decision table where ≥2 conditions decide the outcome → state transitions
  (all-transitions is the default for an HTTP state machine) → the pruned
  heuristic sweep → protocol failure paths. Same pipeline, same order, every
  time: the reviewer re-runs it backwards to find gaps, and the two lists are
  only comparable if the derivation is fixed rather than improvised.
- **Add the failure paths the ACs forgot.** For every endpoint the ticket
  touches, walk the failure-path checklist in
  [references/completeness-rubric.md](../test-reviewer/references/completeness-rubric.md)
  (auth, validation, not-found, conflict, dependency failure, duplicate
  delivery where events are involved) — with the RFC 9110 obligations from
  test-design-techniques.md §7 for what each response must actually assert —
  and add `Type: failure` tests for the applicable ones, marked
  `Verifies: implied` when no AC states them. ACs describe the happy path;
  production incidents live in what they didn't say.
- **Collapse before emitting.** Derived cases are coverage items, not plan
  entries: a validation matrix is one unit table test with its cases listed,
  and the integration layer gets one representative per partition / boundary
  / status code. Anything derived and then dropped is dropped *on the record*
  (a line under **AC issues** or in the summary) — silent pruning is what
  makes a plan uncheckable later.
- Layer assignment rule of thumb for our stack: pure logic and mapping →
  unit (Go table tests); anything crossing a process boundary the repo
  tests with containers — HTTP round-trip, a Kafka message produced or
  consumed, a Spanner row written — → integration/godog (suites run via
  `go test` + `godog.TestSuite{TestingT}`, infra from testcontainers);
  changes to a `contract/` file or shared event type → contract;
  deployment-only concerns → smoke, and say so rather than planning them
  here.
- **Dual-write ACs get dual assertions.** When an action persists to
  Spanner AND publishes to Kafka, the plan must demand both outcomes
  asserted — the row (named back-door step per godog-standards) and the
  consumed event — ideally in one scenario, so partial writes can't pass.
- **A state change is proven by the state, not by the response that claims
  it.** Whenever an AC's outcome is "…and the <thing>'s status becomes X" —
  a transition, a soft delete, a counter increment, anything the next
  request would see — the success case owes an assertion on the state
  *after* the call: re-read it (a GET, a list, a back-door row read) and
  assert X. The response body echoing X is the handler agreeing with
  itself; a handler that returns 200 with the new status and never commits
  passes every response-only test you can write.

  Watch for the asymmetry, because it is the common shape and it looks
  thorough: plans routinely re-read on the **rejection** cases (to show the
  status did *not* change on a 409) and take the response's word for it on
  the **success** cases. That is backwards — the unchanged assertion is the
  cheap one to satisfy accidentally, since not-persisting is exactly what a
  broken write does. If a `Then` re-reads state anywhere in the plan, every
  state-changing success case in that plan needs one too.

## Step 5 — Deliver

- Write the plan to `plans/<TICKET-KEY>.md` in the target repo (create the
  directory if needed). This is the canonical location the reviewer looks in.
- Offer (do not do unprompted): post the plan as a Jira comment on the
  ticket, and/or open an MR adding the plan file.
- End with a short summary: AC count, test count by layer, any AC issues
  raised, and what the reviewer will later check this plan against.
