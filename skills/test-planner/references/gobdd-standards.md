# gobdd authoring standards

Rules for Gherkin this plugin drafts or reviews, targeted at the team's
**internal `gobdd`** framework — a thin wrapper over `github.com/cucumber/godog`
(canonical module `gitlab.com/hillsidetech/pmf-core/gopkgs/gobdd`). Domain: Go
API services in payments, tested as **acceptance** suites (mostly hermetic
fakes/wiremock, not true integration) across per-region PIB deployments.

Scenarios serve two audiences: they read as *behavior specs* to a product
owner, and execute unambiguously through registered step packs.

> Source note: the framework facts below are drawn from the team's gobdd repo
> (its own `CLAUDE.md`, `internal/steps/*`, `example/`) as captured in the
> workspace `docs/REAL-STACK.md`. Scenario-quality and anti-pattern rules that
> carry cited external backing keep their citation; everything gobdd-specific
> is house-standard unless marked.

## The framework (what gobdd is)

- **gobdd wraps godog.** In a step pack, `sc` is `*godog.ScenarioContext` and
  steps register with `sc.Step(regex, method)`. Your godog knowledge applies;
  gobdd adds packaging, per-PIB config, reporting, adapters, and a step
  catalog on top.
- **Runner:** suites run via `gobdd.Run(t, types.Options{JobName, Tags, ...})`
  from a `features/features_test.go`, gated behind an `-integration` flag in
  `TestMain`. Local: `go test ./features -v -args -integration -gobdd.job
  "<pib>"`. There is no `godog.TestSuite{TestingT}` boilerplate in the
  project — `gobdd.Run` is the entry point.
- **gobdd has its own `CLAUDE.md`.** Plans and reviews must ALIGN with it, not
  invent competing conventions. Its three task families: *authoring features*
  (what the planner produces), *configuring runs* (job files), *extending
  gobdd* (new packs/steps). The non-negotiable layout rules below are from it.

## Step packs (the unit of steps)

Steps live in **packs**, not free functions. The canonical shape (copy it):

```go
type Steps struct { s *FeatureState }                 // or `type Pack struct{}`
func NewSteps(s *FeatureState) *Steps { return &Steps{s: s} }
func (st *Steps) Register(sc *godog.ScenarioContext) { // satisfies types.StepPack
    sc.Before(func(ctx context.Context, _ *godog.Scenario) (context.Context, error) {
        return state.WithState(ctx, st.s), nil          // seed state into ctx
    })
    sc.Step(`^the reservation status is '([^']*)'$`, st.assertReservationStatus)
}
```

- State flows through **godog's `context.Context`** (`state.WithState` /
  `state.FromContext`), reset per scenario — not package globals.
- **Built-in packs ship with gobdd**: `api` (HTTP + a `gobddVar` variable
  system with date/time tolerance), `kafka` (request-response over topics),
  `database` (SQL Server / mssql), `spanner`, `riak`, `splunk`, `common`,
  `swagger`. Steps from these are **wired by import** — see the catalog rule.
- **Project-local domain steps** go in a `custompack`, same pattern, registered
  via `config.OptionsBuilder.WithPack`.
- Steps are **small, orthogonal, composable**. A bespoke scenario-specific
  step that can never be reused is a review defect; prefer the cataloged
  generic steps.

## The `@step` catalog (how "wired" is decided)

Every step method carries structured annotations, e.g.:

```go
// @step:phase then
// @step:param want string "Expected status code"
// @step:example Then the response status is 200
func (st *Steps) theResponseStatusIs(ctx context.Context, want int) error
```

`internal/stepdocs/` compiles these into a catalog of every available step
(phrase, phase, params, example). **This catalog — not a grep of the local
repo — is the source of truth for what steps exist.** Consequences:

- **Planner:** draft Gherkin from *cataloged* step phrases and their
  `@step:example` values. Do not invent a phrase that isn't in the catalog
  (built-in packs + the project's custompack) without flagging that a new
  step must be added (a gobdd "extending" task).
- **Reviewer:** a scenario step is **wired** iff its phrase resolves to a
  cataloged step from a registered pack. A step served by the `api`/`kafka`/
  `database` pack is wired *even though it is not defined in the service
  repo* — it comes from the gobdd import. Only a phrase that matches no
  cataloged step (or a `custompack` method that is never registered) is
  unwired.

## Layout (non-negotiable, from gobdd's CLAUDE.md)

- One `.feature` file **per concern**, at `features/`. Concerns are not split
  across files nor bundled together.
- Setup features live in `features/setup/`, each tagged `@setup`, wired from
  the job file's `setup` block; the main suite filters them out (`~@setup`).
- Job files in `features/jobs/<pib>.yaml` — **one per PIB**, never per
  concern/ticket/environment. Only `setup/` and `jobs/` are allowed under
  `features/` (no `api/`, `domain/`, etc.).
- A `features.go` (`package features`) marker + `features_test.go` runner.

## Job configs and PIBs (a coverage dimension)

- `features/jobs/<pib>.yaml` (`version: gobdd/v1`, `kind: Job`) is the
  authority on per-PIB config: `urls` / `ServiceURLs`, `setup`, and adapter
  connections `kafka` (`kafka.endpoints`), `database`, `spanner`, `riak`,
  `splunk`. PIBs are regional: `row` (Rest Of World), `us`, `oc`, `sa`.
- **PIB coverage is a real dimension.** When a ticket targets more than one
  region, "is this AC covered?" includes "for every PIB it targets, or only
  `row`?" The planner should note the PIB scope; the reviewer should flag an
  AC proven for `row` only when the ticket spans regions.

## Tags

Legacy Cucumber grammar — get this right or CI filters silently break:
`~@wip` excludes, `&&` is AND (`@integration && ~@wip`), comma is OR
(`@smoke,@integration`). The modern grammar (`not @wip`, `@a and @b`) is NOT
supported. Set via `-gobdd.tags`, the job's `GoBDDTags` field, or `GOBDD_TAGS`
in CI. `@setup` is reserved for setup features. Scope a run to one concern
with an `@<Concern>` tag rather than a per-concern job file. No ticket-number
tags — AC traceability lives in the plan file's `Verifies:` mapping.

## Scenario quality

- **Declarative, not imperative.** Steps state intent and observable outcomes;
  wire mechanics live in the step pack, once. `When I publish the request` —
  not `When I produce a JSON message to topic attribution.request`.
  [cucumber.io better-gherkin]
- **One behavior per scenario**, ideally one `When`. A failure must be
  attributable.
- **Scenario Outline + Examples** for one behavior across a value table;
  distinct behaviors never share an outline.
- **Reuse the catalog.** Before drafting a new phrase, check the `@step`
  catalog; a near-duplicate of an existing cataloged step is a defect.

## Persistence and messaging shapes (their reality)

- **HTTP** — the `api` pack: send request → assert response field/status/body.
- **Kafka** — the `kafka` pack: **request-response over topics**. Configure a
  service endpoint (from `job.kafka.endpoints`), publish a request message,
  consume the correlated response, assert response fields/headers. Tests run
  against a **fake connection** (hermetic), not a live broker.
- **Database** — the `database` pack over **SQL Server (mssql)**; Spanner is a
  separate adapter. Assert a row exists / a row's field / a row count.
- **Dual-write** — a single scenario can publish a Kafka message AND assert a
  persisted DB row (the Kafka `FeatureState` carries both a Kafka connection
  and a DB client). Both halves are the contract: **a scenario that asserts
  the message but not the row (or the row but not the message) covers only
  half of a two-sided write** — incomplete, not covered. And the failure
  half matters: if the write fails, no event should be published, and vice
  versa.

## Anti-patterns (reject in review)

- Scenario asserting nothing observable (drives the flow, checks only status
  200) — "Unknown Test" smell (completeness rubric).
- A step phrase that resolves to **no cataloged step** (built-in pack or
  custompack), or a `custompack` method never registered on `sc` — unwired,
  counts as NO coverage.
- **Dual-write asserting only one half** (message without row, or row without
  message) — `partial` at best; name the missing half.
- Conjunctive steps (`When I publish the request and read the row`) — split.
- Literal IDs/timestamps that only work in one environment / one PIB;
  scenarios must own their data (create or seed it, uniquely keyed).
- A per-concern job file (`<Concern>.yaml`) — breaks the one-job-per-PIB rule
  and re-enters `before_all`/`after_all` per concern; scope with an
  `@<Concern>` tag instead.
- UI-level language in an API feature (`When I click ...`).
- A `Then` that re-asserts the `Given` (always-true — "Redundant Assertion").
