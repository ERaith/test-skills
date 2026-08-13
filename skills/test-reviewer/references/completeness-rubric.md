# Test-completeness rubric

The source of truth for coverage verdicts. When a review and this document
disagree, this document wins; when this document is wrong, change it by MR —
never by silently deviating in a review.

## Verdicts

- **covered** — an existing, *wired*, passing test asserts the AC's
  observable outcome at the layer the plan assigned. Evidence: file +
  test/scenario name + the assertion.
- **partial** — something real exists but falls short on one axis: wrong
  layer (unit evidence for a planned integration behavior), incomplete
  assertion (asserts 409 but not the error code the AC names), one half of
  a dual-write, or an outcome asserted only indirectly. Always name the
  axis.
- **missing** — no evidence, or only disqualified evidence (unwired
  scenario, `@wip`, failing test, uncited claim).

**SUFFICIENT** requires: no AC `missing`, and no `partial` on an AC whose
gap is behavior-affecting (a missing error-code assertion on a
409-contract AC is behavior-affecting; a missing log-line assertion is
not). Judgment calls are allowed and must be explained in the report.

## The evidence bar

1. **Cite or it didn't happen.** Every non-missing verdict carries a
   citation the reviewer re-found by grep at review time. Un-refindable
   citations are hallucinations; the verdict is `missing`.
2. **Wired means wired.** A godog scenario counts only if every step has a
   matching registration reachable from the suite's `InitializeScenario`.
   Pending/undefined/`@wip` = no coverage.
3. **Passing means passing.** If the suite was runnable and the test
   failed, it is not evidence. If the suite was not runnable (no container
   runtime, no creds), the report says so — static evidence stands but is
   labeled unexecuted.

## Assertion-quality smells (anti-gaming)

Adapted from the peer-reviewed test-smell catalog at testsmells.org
(tsDetect; CASCON 2019 / MSR 2021). A test exhibiting these cannot be
`covered` regardless of its name:

- **Unknown Test** — no assertion at all (drives the flow, asserts
  nothing, passes forever). Scenario equivalent: a Then that makes no
  observable check.
- **Redundant Assertion** — assertions that cannot fail (`expected ==
  expected`, re-asserting the Given). Verifies nothing.
- **Assertion Roulette** (advisory, not disqualifying) — many assertions
  with no attribution; flag it in Notes because a failure can't be
  localized, but it doesn't downgrade the verdict by itself.
- **The house rule that catches most gaming:** the verdict follows what is
  *asserted*, never what the test is *named*. `TestPayRejectsNonPending`
  asserting only `err == nil` covers nothing about rejection.

## Layer matching

The plan assigns each test a layer; evidence must meet it:

- Planned `integration` (behavior crosses a process boundary: HTTP
  round-trip, Kafka message, Spanner row) → evidence must be a wired godog
  scenario (or equivalent containerized Go test). A unit test with mocked
  boundaries = `partial`: it proves the logic, not the crossing.
- Planned `unit` → any layer's evidence accepted (note over-testing
  upward: it's allowed, not encouraged).
- Planned `contract` → evidence is the contract file change plus its
  provider/consumer tests, not a feature scenario.

## Failure-path checklists

House standard (deep-research 2026-08-12 found no externally documented
equivalent to defer to — treat this as ours to maintain). Walk each list
against every endpoint / consumer / write path the diff touches; an
applicable unchecked item is a gap even when all ACs are green. "Applicable"
is a judgment call — record N/A verdicts with a word of why.

**HTTP endpoint:**
- 400 malformed/invalid body (one representative validation case; the full
  matrix belongs in unit tables)
- 401/403 where the route is authenticated/authorized
- 404 unknown resource
- 409 (or the API's convention) for state-machine/conflict violations
- Dependency failure: downstream 5xx AND timeout → mapped error, no hang,
  no partial side effects
- Idempotency where the method claims it (PUT twice = one outcome)

**Kafka consumer:**
- Duplicate delivery → idempotent outcome (exactly one row; assert
  cardinality, not just values)
- Poison message → skipped/DLQ'd, consumer does not stall, messages behind
  it still process
- Event for unknown/expired key → defined behavior, not a crash
- Tolerant reader where the producer is external: unknown fields ignored,
  absent optionals handled

**Spanner persistence:**
- Update-vs-insert pinned by cardinality (`COUNT(*) == 1` after upsert on
  existing key)
- Row assertions via the direct client (back-door), not only via the API's
  own read path — symmetric write/read bugs cancel out otherwise
- Dual-write consistency: the row AND the published event both asserted
  when one action produces both; and the failure half — write fails ⇒ no
  event
- Timestamp/generated columns asserted as changed/present, not exact

## Honesty rules for the report

- Every limitation stated: what didn't run, what couldn't be fetched, what
  was judged N/A and why. Silence must never imply green.
- The report never softens `missing` to `partial` to avoid friction — the
  SDET reading it makes the merge call, and they can only calibrate
  against reports that mean what they say.
