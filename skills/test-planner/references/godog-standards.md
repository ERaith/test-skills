# godog authoring standards

Rules for Gherkin this plugin drafts or reviews, tuned for our domain: Go
HTTP APIs with Spanner persistence and Kafka produce/consume, integration
suites run hermetically with testcontainers (broker + Spanner emulator).

Scenarios serve two audiences: they must read as *behavior specs* to a
product owner, and execute unambiguously through step definitions.

## Runner and layout (how suites are wired)

- **Suites run through `go test`, not the godog CLI** — the CLI is
  officially deprecated. Use `godog.TestSuite` with `Options{TestingT: t}`
  so scenarios execute as Go subtests and one `go test ./...` (with the
  integration build tag) drives everything.
  [github.com/cucumber/godog README]
- Layout per the official API example: Gherkin in `features/`, step
  definitions in a colocated `*_test.go`. Steps do NOT need to live in the
  package root (a common myth — refuted against the godog repo).
- **Steps are methods on a scenario-state struct** (response recorder, last
  Kafka message, direct Spanner client), registered in
  `InitializeScenario`, with state reset per scenario in a `Before` hook.
  godog v0.12+ also supports `context.Context`-based state passing; the
  struct pattern is the official example's and our default.
- **Steps are small, orthogonal, composable** — "just like Unix tools."
  One generic parameterized HTTP step
  (`^I send "(GET|POST|PUT|DELETE)" request to "([^"]*)"$`) beats a bespoke
  step per endpoint. A large scenario-specific step that can never be
  reused is a review defect.

## Scenario quality

- **Declarative, not imperative.** Steps state intent and observable
  outcomes; wire mechanics (headers, JSON bodies) live in the step
  definition, once. `When I pay for the order` — not `When I POST with
  Content-Type application/json to /orders/5/pay`.
  [cucumber.io better-gherkin]
- **One behavior per scenario**, ideally one `When`. A scenario asserting
  create AND pay AND cancel is three scenarios — a failure can't be
  attributed otherwise.
- **Scenario Outline + Examples** for one behavior across a value table
  (validation matrices, state-transition tables). Distinct behaviors never
  share an outline.
- **Background** only for Givens shared by ALL scenarios in the file, kept
  to 1–2 lines.

## Then steps and persistence (house rule — read this one)

Official Cucumber guidance: a `Then` should verify outcomes observable by
the user or an external system, and warns against Then steps that inspect
the database ("resist that temptation"). [cucumber.io gherkin reference]

**Our deliberate, bounded deviation:** for persistence and event tickets,
the stored row / published message IS the externally observable contract —
other systems consume them. So:

- `Then the evaluation row for "u1" shows verified` — allowed: the
  persistence is the behavior under test, and the step name says it's a
  storage assertion (backed by the direct Spanner client, not the API).
- `Then a "user-verified" event is published for "u1"` — allowed: consumed
  from the topic by the test.
- Inspecting the DB to shortcut asserting an *API-level* behavior — still
  banned, exactly as Cucumber says. If the AC is about the response,
  assert the response.

Rationale documented so reviews don't relitigate it: dual-write services
(Spanner row + Kafka event) have two external contracts beyond the HTTP
response, and scenarios must be able to pin both.

## Tags

Organize along the standard axes [VA Gherkin standards]: purpose
(`@integration`, `@smoke`), environment where relevant (`@ci`), and status
(`@wip` for scenarios ahead of implementation — excluded in CI, removed on
completion). Our assignments:

- `@integration` — needs containers (broker, emulator); run by the
  integration job.
- `@smoke` — deployed-environment safe: invariants only, no exact-value
  assertions, creates its own namespaced data.
- No ticket-number tags — AC traceability lives in the plan file's
  `Verifies:` mapping, not in tags that rot.

**Tag expression syntax is godog's legacy grammar** — get this right or CI
filters silently break: `-t "~@wip"` excludes, `&&` is AND
(`"@integration && ~@wip"`), comma is OR (`"@smoke,@integration"`). godog
does NOT document the modern Cucumber grammar (`not @wip`, `@a and @b`).

## Anti-patterns (reject in review)

- Scenario asserting nothing observable (drives the flow, checks only
  status 200) — see the completeness rubric's "Unknown Test" smell.
- Conjunctive steps (`When I create an order and pay for it`) — split.
- Literal IDs/timestamps that only work in one environment; scenarios must
  own their data (create or seed it, uniquely keyed).
- Scenarios with unregistered/pending steps — these count as NO coverage.
- UI-level language in an API feature (`When I click ...`).
- A `Then` that re-asserts the `Given` (always-true assertions verify
  nothing — "Redundant Assertion" smell).
