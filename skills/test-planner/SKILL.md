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

## Step 1 — Gather the ticket

Resolve the argument:

- **Ticket key or URL** → fetch via the Jira MCP tools available in this
  session (use ToolSearch to find them if not loaded; names vary by server —
  look for issue-get/search tools). Pull: summary, description, acceptance
  criteria, issue links, the parent epic, and any linked Confluence page
  (fetch the page — the spec often contains constraints the ticket dropped).
- **Pasted content** → use as-is, but say which context you're missing
  (epic, spec) and ask for it only if the ACs don't stand alone.
- **No Jira access and nothing pasted** → stop and ask for the ticket
  content. Never invent acceptance criteria.

Number the ACs `AC1..ACn` in document order. If the ticket has no explicit
AC section, extract testable statements from the description and present
them as "derived ACs — confirm before relying on this plan."

## Step 2 — Gather the code context

The plan must fit the service it targets. Locate the repo (from the ticket,
or ask once):

- Prefer a local checkout if one exists; otherwise pull context via `glab`
  (`glab repo clone`, or `glab api` for file reads on GitLab).
- Read: the service README, the `features/` directory (existing godog
  scenarios and their tags), and the step definitions (`*_steps.go`,
  `steps/` — wherever `godog` step registrations live). Grep the handlers
  the ticket touches.

Two things this buys: **layer assignment** grounded in what the repo
actually has (unit vs godog integration vs contract), and **step-vocabulary
reuse** — draft Gherkin should use existing steps wherever possible, not
invent near-duplicates (see [references/godog-standards.md](references/godog-standards.md)).

## Step 3 — Testability triage (do this BEFORE planning)

Judge each AC against: observable outcome stated? inputs/preconditions
stated? unambiguous pass/fail? If an AC fails the bar (e.g. "search should
be fast", "handles errors gracefully"), do not write vague tests around it —
flag it at the top of the plan under **AC issues**, with a concrete rewrite
suggestion ("propose: p95 search latency < 300ms at 100 rps" style). If most
ACs are untestable, stop and report back instead of emitting a vacuous plan:
pushing ambiguity back to the ticket before code exists is this skill's
highest-value output.

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
- **Add the failure paths the ACs forgot.** For every endpoint the ticket
  touches, walk the failure-path checklist in
  [references/completeness-rubric.md](../test-reviewer/references/completeness-rubric.md)
  (auth, validation, not-found, conflict, dependency failure, duplicate
  delivery where events are involved) and add `Type: failure` tests for the
  applicable ones — marked `Verifies: implied` when no AC states them. ACs
  describe the happy path; production incidents live in what they didn't say.
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

## Step 5 — Deliver

- Write the plan to `plans/<TICKET-KEY>.md` in the target repo (create the
  directory if needed). This is the canonical location the reviewer looks in.
- Offer (do not do unprompted): post the plan as a Jira comment on the
  ticket, and/or open an MR adding the plan file.
- End with a short summary: AC count, test count by layer, any AC issues
  raised, and what the reviewer will later check this plan against.
