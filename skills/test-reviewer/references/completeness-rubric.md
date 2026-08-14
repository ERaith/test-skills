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
2. **A double-quoted string is a promise that those exact bytes are in the
   diff.** House standard, and the rule that makes rule 1 checkable rather
   than merely asserted. Inside an Evidence cell, `"..."` is reserved for
   text copied character-for-character out of a file under review — a
   scenario title, a step, an error code, an assertion string. Therefore:
   - **Never elide, never reflow, never re-word.** `"...no guest name field
     is rejected"` and a plan sentence tidied from `code "invalid_date"` into
     `"with error code invalid_date"` are both un-greppable, and a reader who
     greps for them concludes you invented them. If the exact string is long
     or awkward, cite `file:line` and drop the quotation marks entirely — a
     bare `file:line` claims nothing verbatim and so cannot break this rule.
   - **Text that is not from the diff never takes double quotes.** Output you
     observed from a command, a skeptic agent's wording, your own paraphrase:
     all real evidence, none of it a citation. Introduce it in prose or in
     backticks and say where it came from — `run output: expected status 400,
     got 201`, or *the skeptic's residual was that it had not executed the
     suite itself*. Reserving one mark for one meaning is what lets a
     reviewer of the reviewer check the whole column with `grep -F`.
3. **Wired means wired.** A godog scenario counts only if every step has a
   matching registration reachable from the suite's `InitializeScenario`.
   Pending/undefined/`@wip` = no coverage.
4. **Passing means passing.** If the suite was runnable and the test
   failed, it is not evidence. If the suite was not runnable (no container
   runtime, no creds), the report says so — static evidence stands but is
   labeled unexecuted.

## Confidence (per verdict)

Every verdict carries a confidence: `high` / `medium` / `low` — the same
three buckets, spelled the same way, as the `test-planner` CLI
(`ConfidenceSchema` in `src/schemas.ts`; coarse buckets because models
calibrate badly on continuous scores and structured output can't range-check
a float). A skill report and a CLI report are therefore comparable
verdict-for-verdict. Bucket calibration follows *LLM-as-a-Judge for Scalable
Test Coverage Evaluation* (arXiv:2512.01232); the derivation table below is
house-standard.

Confidence answers **"how sure are we this verdict is right?"** — not "how
good is the coverage". A `missing` verdict is usually `high` confidence.
Confidence never changes a verdict and never changes SUFFICIENT /
INSUFFICIENT; it tells the SDET where to spend their own eyes.

**Confidence is derived, not felt.** Compute it from the two structural
properties below. Never adjust it because the change looks careful, the
author is senior, the diff is small, or the report feels harsh.

- **executed** — the test was actually run during this review and passed
  (`go test ./...`, or the `@integration` godog suite against a live
  container runtime). Evidence read but not run is **static-only**.
- **skeptic-survived** — the `covered` verdict went through an explicit
  refutation attempt (SKILL.md Step 3b — an independent skeptic agent per
  verdict where a subagent runtime exists, the same attacks inline where it
  does not) that failed to break it. A refutation carries the same citation
  bar as coverage: an objection that cannot be re-found does not refute, it
  caveats.

| Verdict | `high` | `medium` | `low` |
|---|---|---|---|
| `covered` | skeptic-survived **and** executed | static-only evidence, or survived refutation with a caveat you could not fully rule out | the AC→test mapping needed interpretation, or a step's wired-ness could not be confirmed |
| `partial` | the shortfall axis is mechanically checkable — a named assertion is absent from a file you read, or the layer contradicts the plan | the shortfall took judgment (does asserting 200 satisfy "returns the order"?) | you are unsure whether the right reading is `partial` or `missing`; say which way you leaned |
| `missing` | the whole evidence surface was inventoried — both sources enumerated, greps re-run — and nothing exists | part of the surface was unavailable (repo tree truncated, only the MR diff readable, step registrations unreadable) | inputs themselves were missing: ACs not fetched from the ticket system, or the repo could not be read |

Caps, applied after the table — lowest wins:

1. **No `high`-confidence `covered` without an execution.** Static-only
   evidence caps `covered` at `medium`, however clean the code reads.
2. **Un-refindable citations are not a confidence problem.** A citation you
   cannot re-find by grep makes the verdict `missing` at `low` confidence —
   the same downgrade the CLI's normalizer applies to hallucinated evidence.
3. **Ad-hoc plan caps the overall report** (below) at `medium`, and caps any
   verdict whose reasoning leaned on a layer assignment at `medium` — you
   invented that layer assignment minutes ago; it was never agreed.

### What would raise it

Every verdict below `high` carries a one-line **"what would raise it"**: the
specific, executable action that would move this verdict up a bucket, naming
the thing. "Run the `@integration` suite (needs a container runtime)" or
"assert `data.error.code`, not only the 409 status" — never "add more tests"
or "investigate further". If nothing could raise it (the AC is genuinely
ambiguous and needs product input), say that instead and name who decides.

### Overall report confidence

The lowest per-AC confidence in the report, then capped by:

- reviewed against an ad-hoc plan (no `plans/<KEY>.md`) → cap `medium`
- suite not runnable (no container runtime, no creds) → cap `medium`
- ACs not obtained from the ticket system (read off the MR description) →
  cap `low`

State it on the verdict line. A `SUFFICIENT` resting on any `low`-confidence
`covered` verdict must name that AC on the verdict line — a green report the
reviewer half-believes is worse than a red one, because nobody re-checks it.

**State a systemic limitation once.** If a limitation applies equally to
every verdict — the suite could not run, the repo has no step-definition
layer to inspect, the ticket system was unreachable — it belongs on the
verdict line as an overall cap, not repeated as a per-verdict downgrade.
Per-verdict confidence exists to discriminate *between* verdicts; a column
reading `low, low, low, low` tells the SDET nothing about where to look.

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

*Which* paths we require is house standard (deep-research 2026-08-12 found no
externally documented equivalent list to defer to — treat the selection as
ours to maintain). *What each one means*, and what a conforming response must
contain, is RFC 9110 — cited per line below, with the full obligation table
in
[test-design-techniques.md](../../test-planner/references/test-design-techniques.md)
§7. Walk each list against every endpoint / consumer / write path the diff
touches; an applicable unchecked item is a gap even when all ACs are green.
"Applicable" is a judgment call — record N/A verdicts with a word of why.

**HTTP endpoint:**
- 400 malformed/invalid body [RFC 9110 §15.5.1] (one representative
  validation case; the full matrix belongs in unit tables). Where the body
  parses but the instructions can't be processed, 422 is the distinct code
  [§15.5.21] and a wrong media type is 415 [§15.5.16] — an API collapsing
  all three into 400 is a finding, not a shortcut.
- 401/403 where the route is authenticated/authorized [§15.5.2, §15.5.4].
  A 401 **MUST** carry `WWW-Authenticate` [§15.5.2] — assert the header, not
  only the status.
- 404 unknown resource [§15.5.5]; 405 for a known method the resource does
  not support, which **MUST** carry `Allow` [§15.5.6].
- 409 (or the API's convention) for state-machine/conflict violations
  [§15.5.10]
- Dependency failure: downstream 5xx AND timeout → mapped error [§15.6.4,
  §15.6.5], no hang, no partial side effects
- Idempotency where the method claims it — PUT and DELETE are idempotent by
  [§9.2.2], so twice = one outcome, asserted by *cardinality* (one row, one
  event), not by a second 200. POST is not idempotent: what a duplicate
  submission does must be *defined* somewhere, and the review says which.

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
