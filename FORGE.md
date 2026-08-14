# Forge queue: test-skills maturation

Long-running iterative work list, executed by a looping Claude Code session
on the homelab (deepthought, ~/forge/). Each iteration: pick the topmost
unfinished item (or the most valuable follow-up discovered last iteration),
do ONE meaningful unit of work, verify it, commit+push THIS repo, and append
a dated entry to the Log below (the log lives in this file — committed, so
progress is visible from any machine via git pull). Small, verified,
committed steps — never a big broken WIP.

## Environment (deepthought)

- This repo: `~/forge/test-skills` (github.com/ERaith/test-skills — push
  after every completed item; gh is authed).
- CLI fixtures: `~/forge/test-planner` — **READ-ONLY reference** (no
  remote; copied from the laptop). Contains
  `fixtures/tickets/PROJ-101.json`, `fixtures/go-api/features-full/`,
  `fixtures/go-api/features-partial/`, and `plans/` examples.
- Golden eval repo (queue item 5): create at `~/forge/test-skills-eval`,
  **local git only — do not create a GitHub repo or push it anywhere.**
- Go toolchain: `~/.local/go/bin` (on PATH); container runtime: docker.

## Ground rules

- Advisory only — never add merge gating, never touch GitLab settings.
- Plugin must validate (`claude plugin validate ~/forge/test-skills`)
  before every push.
- Plan format stays byte-compatible with the test-planner CLI fixtures.
  Verdict vocabulary stays covered/partial/missing.
- Rubric/standards changes must cite a source or be marked house-standard.
- Evals are ground-truthed: every eval fixture has a documented expected
  verdict BEFORE the reviewer runs against it.
- Touch nothing outside: `~/forge/test-skills`, `~/forge/test-skills-eval`,
  and reads of `~/forge/test-planner`.
- The laptop's planning docs are NOT here — do not try to update them.
  Anything that belongs in planning/{PLAN,RESEARCH}.md goes in the Log
  under a **For planning sync:** marker; the laptop session folds those in
  after the forge run.
- Stop condition: queue empty and last full eval sweep stable across 3
  consecutive runs, or ~24h elapsed — then write a final summary Log entry,
  push, and end the loop.

## Priority & process (revised 2026-08-14)

Planner and reviewer are built **hand in hand, rung by rung** — the plan file
is their shared interface, so each rung exercises BOTH. The pain we solve is
ticket volume -> tangible testable plans fast (planner), with the reviewer
checking MRs against those plans. Per-rung cycle:

  1. Build the rung: scenario + mixed-quality tickets + ground truth.
  2. Test BOTH skills: planner drafts plans; reviewer verdicts the MRs.
  3. Produce results on the TARGET RUNTIME MODEL (Sonnet-class): planner
     ship/rewrite/wrong + did-it-flag-bad-ACs; reviewer per-AC accuracy.
  4. Update planner OR reviewer based on what the results show is missing.
  5. MERGE TO MAIN — each rung is a merge-to-main milestone.
  6. STOP. Human + Claude jointly decide the NEXT complexity increment
     before the next rung. No autonomous climbing past a rung.

Evidence rules: baseline before optimizing; every change justified by a
measured eval delta; real SDET tickets replace synthetic ones as they arrive
(synthetic tickets encode our assumptions — always flag them). Deferred until
a rung's results demand it: fan-out as the default runtime path
(calibration/high-assurance only), the SHA-keyed step catalog, and gating.
The skeptic hold still governs whether a rung PASSES (PROVING-GROUND.md).

## OVERNIGHT DIRECTIVE (2026-08-14 night — autonomous, no human present)

Go DEEP on R1, not WIDE to R2. Allowed autonomous work, in priority order:
  1. Converge R1 to a genuinely PASSED state: build the rung, eval BOTH
     skills, fix whichever (planner/reviewer) the scorecard shows is weak,
     re-eval, repeat until the advance gate + skeptic hold pass. Merge each
     improvement to main. Log every calibration change and every scorecard.
  2. Eval-methodology research (well-defined, needs no human): how teams
     validate LLM judges/graders — gold-set construction, inter-rater
     agreement, calibration, LLM-as-judge failure modes. Fold findings into
     PROVING-GROUND scoring as cited commits.
HARD STOP: do NOT build R2 or ANY new-complexity rung (Spanner/Kafka/3-svc/
mess-batch). New complexity is a HUMAN decision at tomorrow's checkpoint.
If R1 passes and research is done, idle — do not invent new scope.

## Queue

- [x] 1. Agent fan-out in both skills (context/evidence lanes + skeptic).
- [x] 2. Per-AC confidence in the reviewer report.
- [x] 3. test-design-techniques.md reference (ISTQB/Hendrickson/RFC 9110),
      wired into planner Step 4 + reviewer Step 3.
- [~] 4. Reviewer CLI-fixture baseline — SUPERSEDED by R1 (which exercises
      both skills on a richer scenario). Skip unless a quick reviewer-only
      sanity number is wanted first.

### Rungs — one at a time, human checkpoint between each

- [~] R1. **Single HTTP service** — built + evaluated (heavy model); PROVISIONAL pass. Re-run on Sonnet to certify. Original scope: Build the scenario + mixed-quality
      tickets (one crisp, one vague/untestable AC, one missing failure
      paths) + ground truth OUTSIDE the eval repo. Planner drafts plans;
      reviewer verdicts the seeded branches (full / AC-missing / gamed-test
      / unwired). Scorecard BOTH. Merge to main. STOP for the R2 decision.
- [ ] R2. **+ Spanner emulator** (DECIDED 2026-08-14) — persistence-as-contract: back-door row assertions, update-vs-insert cardinality, first testcontainer. Builds AFTER the Sonnet re-run confirms R1.
- [ ] R3+. Decided jointly after each rung — not pre-committed. The
      complexity menu to pick the next increment from: +Spanner emulator,
      +Kafka produce/consume, +3 microservices, the mess batch
      (contradictory ACs, ticket-vs-MR scope mismatch). Choose the next
      increment from what R1 reveals is weak, not from a fixed sequence.

## Log

### 2026-08-14 — R1 (single-service) built + evaluated; provisional pass

Built rung1 (reservations-api): spec + 3 mixed-quality tickets (RES-102: 2
crisp ACs, 2 untestable [AC2 "gracefully", AC4 "responsive"], 1 implied
[double-click idempotency]) + 4 seeded-defect branches + perturbations +
ground truth OUTSIDE the eval repo. Both skills, 5 clean-room runs/arm.

Results (ON THE HEAVY BUILD MODEL — caveat 1):
- Planner (RES-102): flagged both untestable ACs with SOURCED rewrites and a
  spec-open-question refusal; added 3 implied failure-path tests; all ACs
  covered; stored-status back-door discipline; clean format. Pass.
- Reviewer: per-AC verdicts matched EXPECTED on all 4 arms; correct overall
  SUFFICIENT/INSUFFICIENT; caught the gamed branch's assertion-free test
  (Unknown Test smell) with line-level evidence; 6 skeptics survived. One
  soft miss (T7 partial->covered, full arm). Calibrated over 6h — 6 reviewer
  commits (citation pre-flight, greppable citations, skeptic-in-scope).

CAVEATS (gate a real pass):
1. Ran on the heavy build model, NOT the Sonnet runtime tier. run-cleanroom.sh
   now pinned to `--model sonnet`; R1 must be RE-RUN on Sonnet to certify.
2. Ground truth builder-authored; walked through with the human this session
   (RES-102 plan + gamed-branch verdict) -> proceeding to R2 = provisional
   acceptance. Explicit AC-by-AC sign-off not separately recorded.
3. Run hit the subscription session limit and spun ~600 useless iterations;
   forge.sh now HALTS on session limit (fixed this session).

DECISIONS:
- R1 = PROVISIONAL pass, pending the Sonnet re-run.
- R2 = + Spanner emulator (persistence-as-contract; back-door row assertions,
  update-vs-insert cardinality). Builds AFTER the Sonnet re-run confirms R1.

NEXT: Sonnet re-run of R1 (quota resets 15:20 UTC) -> human review -> R2.


*(one dated entry per iteration: what was done, what was learned, what
changed in the queue)*

### 2026-08-13 — queue item 2: per-verdict confidence

**Done.** Confidence is now a first-class part of the reviewer output.

- `completeness-rubric.md` gains a **Confidence (per verdict)** section: the
  `high`/`medium`/`low` buckets (same spelling as the CLI's
  `ConfidenceSchema`, so reports are comparable verdict-for-verdict), a
  derivation table for all three verdicts × three buckets, three caps, the
  "what would raise it" requirement, and overall-report confidence.
- `test-reviewer/SKILL.md` gains **Step 3b** (refute every `covered` before
  keeping it — outcome vocabulary refuted / survived-caveat / survived) and
  **Step 3c** (derive confidence). Step 2 now requires tagging each piece of
  evidence executed-vs-static-only *as it is inventoried*. Step 4's report
  shape gains a Confidence column on both tables, a "what would raise it"
  block, an overall confidence on the verdict line, and a
  `n high / n medium / n low` tally in Notes (mirrors the CLI's telemetry
  `confidenceCounts`, so eval sweeps can diff them).

**Design decisions, and why.**

1. **Structural, not felt.** The CLI asks the model how confident it feels
   (`prompts.ts`: "high = unambiguous"). That is self-report, and self-report
   is what item 6 will have to calibrate away. Here confidence is *derived*
   from two recorded facts — was it executed, did it survive refutation —
   so two runs over the same evidence should produce the same confidence
   even if the prose differs. Stability is the point.
2. **No `high`-confidence `covered` without an execution.** The single
   strongest cap. Static reading of a test is a hypothesis about what it
   does.
3. **Un-refindable citation is not a low-confidence `covered`** — it is
   `missing` at `low`. Deliberately mirrors the CLI normalizer
   (`verify.ts` `checkEvidence`: drop hallucinated evidence, downgrade
   verdict, set confidence `low`), so the two tools fail the same way.
4. **Systemic limitations are stated once, on the verdict line.** Found
   while dry-running (below): naively applying the table to a Gherkin-only
   corpus made every verdict `low` for the same reason, which destroys the
   column's discriminating power. A limitation that applies to every verdict
   equally is an overall cap, not four downgrades.
5. **Confidence never moves SUFFICIENT/INSUFFICIENT.** A `low`-confidence
   `covered` is still `covered`; it just gets named on the verdict line so
   the SDET knows where to spend their own eyes.

**Verified** — plugin validates (`claude plugin validate`, passes with the
intentional no-version warning), plus a paper dry-run of the new rules
against the ground-truthed CLI fixtures (`~/forge/test-planner`,
`tickets/PROJ-101.json`):

| | AC1 | AC2 | AC3 | AC4 | overall |
|---|---|---|---|---|---|
| features-partial, hand-applied | covered/medium | missing/high | partial/high | missing/high | INSUFFICIENT, medium |
| CLI `reports/PROJ-101.json` | covered/high | missing/high | partial/high | missing/high | fail |
| features-full, hand-applied | covered/medium | covered/medium | covered/medium | covered/medium | SUFFICIENT, medium |

Verdict vector matches the CLI exactly on the partial set, and the expected
SUFFICIENT/INSUFFICIENT split holds. The one divergence is informative: the
CLI calls AC1 `high` where the derived rule says `medium`, because the CLI
never executes anything — every CLI `high` on a `covered` verdict is
structurally unearnable under this rubric. Rules proved decidable (no
verdict needed a coin-flip) and the column discriminates (medium/high/high/
high, not four identical values).

**Dependency handled.** Item 2's `skeptic-survived` input needs item 1's
skeptic agents, which are not built yet. Rather than block, Step 3b defines
the refutation pass inline *and* fixes the outcome vocabulary as the
contract item 1 will implement against — item 1 changes the mechanism
(one parallel agent per `covered` verdict) without touching the confidence
rules.

**Learned / for the next iterations.**

- The fixture corpus has no step-definition layer at all, so the rubric's
  wired-means-wired bar cannot be exercised against it. Item 4 must treat
  "unwireable corpus" as a known skill-vs-CLI divergence and not a rubric
  bug; the wiring bar gets its real test on the item 5 golden repo, which
  should include an unwired-scenario branch (it already does).
- Item 6 now has a second axis worth measuring: not just verdict
  accuracy/stability, but *confidence* stability — the derived rules predict
  it should be near-deterministic. If it is not, the derivation has a hole.

**For planning sync:** the skills now diverge from the CLI deliberately on
confidence semantics — CLI = model self-report, skills = structural
derivation. Worth calling out in RESEARCH.md D1: the structural definition
is what makes agent confidence auditable, and therefore what a Phase-3
gating carve-out could actually key on ("gate only on high-confidence
verdicts") — self-reported confidence could never carry that weight.

### 2026-08-14 — queue item 1: agent fan-out

**Done.** Both skills now fan out, and the rules for doing so live in one
new shared reference, `skills/test-planner/references/agent-fanout.md`
(planner-side by the same convention item 3 uses for shared docs; the
reviewer links it through `${CLAUDE_PLUGIN_ROOT}`).

- **Planner Step 1** dispatches the context wave: **A1** ticket graph (the
  ticket verbatim, its epic, and every linked/blocking/dependent ticket —
  including ACs in a *linked* ticket that constrain this one), **A2**
  Confluence spec (specifically: what the spec says that the ticket does
  not), **A3** repo suite inventory (scenarios+tags, every step
  registration verbatim, how the integration suite is executed, unit-test
  layout). New **Step 2** merges the wave; old Steps 3-5 keep their
  numbers.
- **Reviewer Step 2** dispatches the evidence wave: **B1** unit tests
  (what each test *asserts*, quoted), **B2** feature files (every step
  verbatim), **B3** step wiring (patterns, bindings, reachability, and
  what each binding actually touches).
- **Reviewer Step 3b** dispatches one skeptic agent per `covered`
  candidate, each given one AC, one citation, the repo path and the attack
  list — and nothing else.

**Design decisions, and why.**

1. **Agents gather, the caller decides.** A lane returns facts with
   citations — never a verdict, a confidence or an AC id. Three reasons,
   in order: (a) traceability — a verdict assembled from a subagent's
   *conclusion* has no citation the caller ever saw, which launders an
   unverified claim into the report and quietly voids the rubric's
   cite-or-it-didn't-happen bar; (b) stability — enumeration reproduces
   across runs, judgment distributed over N sampled sessions does not;
   (c) one rubric — the rubric lives in the caller's context, so a judging
   subagent would either judge without it or need it pasted per lane,
   where it forks. The skeptic lane is the single deliberate exception,
   and even there the caller applies the outcome.
2. **B3 is blind to B2, and the caller does the matching.** The most
   gameable judgment in the whole review is wired-means-wired. One agent
   that reads a scenario and *then* hunts for its steps accepts a
   near-match, because it knows what it is hoping to find. B3 is never
   told which scenarios exist; it inventories registrations. The caller
   matches B2's verbatim step strings against B3's pattern list. This is
   the main reason the lanes are split the way item 1 specified them
   rather than "one agent per evidence source".
3. **The evidence bar is symmetric.** A refutation is a claim too. A
   skeptic's `refuted` only moves a verdict if the caller can re-find its
   citation; an uncitable objection becomes `survived-caveat`. Without
   this, fan-out just adds a machine for downgrading verdicts on vibes —
   the mirror image of the optimism the rubric already refuses.
4. **Every agent citation is re-grepped before it reaches output.**
   Fan-out multiplies the number of claims; it must not multiply the
   number of unchecked claims. A lane finding that cannot be re-found is
   discarded and the lane drops to `partial`.
5. **AC numbering happens once, in the caller, from A1's verbatim block.**
   No lane numbers or renumbers ACs. `AC<n>` ids that shift between runs
   make every downstream verdict incomparable, which would wreck the
   item 5-7 scorecards before they start.
6. **Verbatim prompt templates, bounded scope per lane.** An improvised
   "look around and tell me about testing" prompt is exactly how two runs
   of one review return different evidence sets. Fan-out is a stability
   *risk* as much as a quality gain, and the controls are all in the
   reference.
7. **Injection guard on every lane prompt** — repo/ticket/spec content is
   untrusted data, never instructions. Mirrors the CLI's `VERIFY_SYSTEM`
   guard (`src/lib/prompts.ts`); a fanned-out reviewer has more mouths to
   feed poisoned Gherkin to than a single session did.
8. **`not-found` is mandatory envelope output.** "No registration matching
   `^a task exists$`" is what produces a `missing` verdict. A lane that
   reports only what exists biases every downstream verdict green.
9. **No confidence semantics changed.** Item 2 wrote its
   refuted/survived-caveat/survived vocabulary *as the contract item 1
   would implement against*, and that held exactly — Step 3b's mechanism
   swapped with no edit to the rubric's derivation table or caps. The
   rubric's `skeptic-survived` definition gained one clause (independent
   agent where a runtime exists, the same attacks inline where it does
   not, refutations held to the citation bar). Whether independent
   skepticism actually flips verdicts that inline skepticism keeps is left
   as a *measured* question for rung 1 rather than priced in as a cap
   invented here.
10. **The fallback never skips a lane.** No subagent runtime → do every
    lane inline, in lane order, same scope, same envelope, and say so (in
    the plan summary / on the verdict line). The wave structure is about
    coverage of surfaces; parallelism is only the speed-up.

**Verified.** No Go in this repo, so build/test are N/A; the checks that
apply:

- `claude plugin validate` passes (the one intentional no-version warning).
- All 8 reference links across both SKILL.md files resolve, including the
  two new `${CLAUDE_PLUGIN_ROOT}` ones (script-checked, 0 broken).
- `plan-format.md` byte-identical — `git diff --stat` on it is empty, so
  CLI plan compatibility is untouched.
- **Lane scopes checked against the real read-only fixtures**
  (`~/forge/test-planner/fixtures`): 7 `.feature` files, **0** step
  registrations (`ctx.Step(`/`s.Step(`/`InitializeScenario`/
  `godog.TestSuite`), **0** `*_test.go`. So on that corpus B2 = `ok`,
  B1 and B3 = `unavailable` — which is precisely the case the reference
  documents as "scenarios exist, no step layer readable ⇒ no scenario can
  be counted as evidence". Item 2's log predicted this corpus limitation
  on paper; it is now measured.
- **The envelope grammar and the B2×B3 merge rule were executed, not just
  described.** Wrote a parser to the documented grammar plus the caller's
  merge rule, generated the B2 envelope *mechanically from the real
  fixture feature file* (16 step findings, Background steps attributed per
  scenario), and ran two cases. Case 1 (real corpus, B3 `unavailable`):
  all 3 scenarios → not-evidence, reason "no step layer readable". Case 2
  (synthetic B3): `wired` for the two scenarios whose every step matches a
  substantive binding, and not-evidence for each of the four defect
  branches with the right reason — `@wip`, unmatched step, registration
  unreachable (`registered in (unreached)`), and pending binding
  (`touches: nothing`). Five branches, five distinct correct reasons.
- **The skeptic rules were executed too**: 5 skeptic blocks parsed against
  the documented shape, and the symmetric-evidence-bar logic gives
  refuted → downgrade only when the citation is re-findable; refuted with
  `evidence: none` → `survived-caveat`; refuted with a citation the caller
  cannot re-find → `survived-caveat`; survived-caveat → covered/medium;
  survived → covered. 5/5 as documented.
- Harness lives in `/tmp/fanout-check/` (throwaway — not committed; it
  tests a documented contract, and its permanent home is the rung-1
  scorecard tooling, not this repo).

Deliberately **not** run: the lanes themselves, as live agents, from this
session. PROVING-GROUND rule 1 forbids the session that built a thing from
being the session that grades it, and a fan-out I dispatched here with the
answers already in context would measure nothing. The lanes get their real
first exercise in the item 4 smoke eval and their graded one at rung 1,
both clean-room.

**Learned / for the next iterations.**

- Item 1 as specified had a hidden ordering dependency: A2's input (spec
  URLs) comes from A1's output, so "all lanes in one message" is false for
  the planner. Resolved explicitly — A1+A3 together, A2 joins them if a
  URL is already in hand, else it goes the instant A1 returns. Worth
  watching for the same shape at rung 4, where cross-service evidence may
  want a second B-wave seeded by the first.
- Flat `findings` lines lose structure when what the caller needs is a
  *mapping* (scenario → its steps). B2/B3 therefore got fixed
  finding-line shapes with the owner and the verbatim string in known
  positions. Discovered by writing the parser: my first B2 envelope parsed
  into an unusable flat list. Any future lane whose output is merged
  mechanically needs its line shape pinned the same way.
- The three reviewer lanes give rung-1 debugging a free tool: if verdicts
  flip between the 3 required runs, diff the lane envelopes — identical
  envelopes ⇒ caller/rubric problem, differing envelopes ⇒ lane-scope
  problem. Envelopes should be kept as run artifacts in the scorecard
  (noted in the reference; item 5 should implement it).

**For planning sync:** the fan-out makes the reviewer's most gameable
judgment — wired-means-wired — *structurally* independent: the agent that
inventories step registrations never sees the scenarios it might be
tempted to match them to, and the agent that attacks a `covered` verdict
never sees the verdict list it might calibrate to. Together with item 2's
structural confidence, that is the concrete D1 answer to "what would make
an agent verdict auditable enough to gate on": not a better prompt, but an
architecture where the load-bearing judgments are made by parties that
cannot see each other's expectations, and every claim in either direction
carries a citation the caller re-found.

### 2026-08-14 — queue item 3: test-design-techniques reference

**Done.** New shared reference
`skills/test-planner/references/test-design-techniques.md`: how to derive the
cases an AC implies but does not say. Read by both skills, in opposite
directions — planner Step 4 forward (AC → coverage items → planned tests),
reviewer Step 3 backward (same derivation, then diff against the evidence).

- **A fixed seven-step pipeline**, not a menu: variables/outcome → equivalence
  partitioning → BVA → decision table → state transition → heuristic sweep →
  protocol failure paths. Steps 2–5 are ISTQB black-box techniques with their
  coverage criteria quoted from CTFL v4.0.1 §4.2.1–§4.2.4; step 6 is the
  Hendrickson/Lyndsay/Emery cheat sheet pruned to this domain (checklist-based
  testing, §4.4.3); step 7 is RFC 9110.
- **Worked examples are the real fixtures**, not invented ones: the EP /
  decision-table example is the tasks API (`~/forge/test-planner`), the
  state-transition example is the orders API's pending/paid/cancelled machine
  — 3 states, 2 valid transitions, 6 all-transition cells.
- **Wired in.** Planner Step 4 gains the derivation bullet plus a *collapse
  before emitting* rule; reviewer Step 3 gains the edge-case gap check; the
  rubric's HTTP failure-path checklist now carries per-line RFC 9110 section
  citations and the two `MUST`s.

**Design decisions, and why.**

1. **Order is load-bearing, and fixed.** BVA needs EP's partitions; the
   heuristic sweep is last so it only catches what the models structurally
   cannot see. Steps 2–5 enumerate from a model and therefore reproduce across
   runs; step 6 does not, which is exactly why it is a bounded table with
   named attacks rather than "think about edge cases" — §4.4.3's own warning
   is that checklist items which are too general are worthless.
2. **Derived items are gaps, never verdicts** (the load-bearing rule). The
   derived set is always larger than the AC set, so if a derived miss could
   downgrade an AC, every review would be INSUFFICIENT and the verdict column
   would stop discriminating — it would mean "the reviewer ran its checklist".
   The one exception is precise: when the *AC itself* names the item (it
   states the 200-char limit, it names the 409), the item is part of the AC's
   observable outcome and its absence is `partial`. Measured below: the
   derived gap list is **identical (9 gaps) on both the full and the partial
   fixture branch** — a column that cannot tell those two suites apart has no
   business touching the verdict.
3. **All-transitions coverage by default for HTTP state machines** (house
   standard; §4.2.4 asks it only of mission/safety-critical software). On a
   public API an invalid transition is not exotic — it is what a retry, a
   double-click or a competing consumer produces, and the answer to it is a
   contract. The orders fixture makes the cost of the weaker default concrete:
   pay + cancel on a pending order is 100% valid-transitions coverage and 33%
   all-transitions, with the entire 409 contract untested behind a
   green-looking suite.
4. **RFC 9110 grounds the meaning, not the menu.** Which failure paths we
   require stays house standard; the RFC fixes what each code means and what
   the response must contain. Two are directly assertable `MUST`s —
   `WWW-Authenticate` on 401 (§15.5.2), `Allow` on 405 (§15.5.6) — so those
   became "assert the header, not only the status". Same for idempotency
   (§9.2.2): PUT/DELETE owe a *cardinality* assertion, and POST — explicitly
   not idempotent — owes a **defined** duplicate-submission behaviour, since
   second-create / 409 / idempotency-key are all legitimate and only one is
   implemented.
5. **The derivation adds no plan syntax.** No `Technique:` field — the plan
   stays byte-compatible with the CLI, so attribution lives in the test title
   and in the reviewer's gap lines. Two extra rules keep the derivation from
   exploding the plan: collapse to the layer (a 12-row matrix is one unit
   table test; integration gets one representative per partition/boundary/
   code), and Each Choice rather than Cartesian combination (§4.2.1, citing
   Ammann 2016).
6. **Nothing derived may vanish silently.** Every partition, boundary,
   feasible column and transition cell ends up either mapped to a test id or
   written down as a deliberate omission — otherwise the reviewer cannot check
   the planner's work, which is the whole point of both skills sharing one
   pipeline.

**Verified.** No Go in this repo (build/test N/A). What was actually run:

- `claude plugin validate` passes (the one intentional no-version warning);
  14 reference links across the skills resolve, 0 broken; `plan-format.md`
  byte-identical (`git diff --stat` empty on it).
- **Every citation machine-checked against the primary source, not memory.**
  Downloaded RFC 9110 and the CTFL v4.0.1 syllabus PDF; a script extracted all
  17 RFC section numbers cited across the skills and confirmed each exists
  **and** that its heading matches the code it is cited for (§15.5.6 = 405
  Method Not Allowed, etc.) — 17/17. Seven quoted RFC claims (both `MUST`s,
  the idempotency definition, PUT/DELETE idempotent, 201-Location, the 400 and
  422 definitions) matched verbatim, 7/7. Both ISTQB block quotes matched the
  syllabus verbatim after whitespace normalisation, plus 15 further syllabus
  claims (EP coverage incl. invalid partitions, Each Choice/Ammann, 2-value
  Craig/Myers, decision-table coverage, all three state-transition criteria,
  defect masking, Brykczynski's checklist caution). Two initial mismatches
  were **real citation defects and were fixed**: the EP quote had silently
  dropped a parenthetical, and the 3-value BVA example had the wrong
  mis-implementation (the syllabus's is `if (x ≤ 10)` implemented as
  `if (x = 10)`, which 2-value data 10/11 cannot detect and x=9 can — my first
  draft wrote `if (x < 10)`, which 2-value *does* catch).
- **The worked examples were executed against the running fixtures.** Built
  and ran both read-only fixture APIs from a `/tmp` copy. The orders state
  table was exercised cell by cell — all 6: `pending`+pay/cancel → 200 with
  the state changed, and all four invalid cells → 409 `invalid_transition`
  with the state unchanged. The published table is measured behaviour.
- **The tasks-API partition findings are live, not paper.** Probed 14 requests:
  five *distinct* invalid partitions (empty title, absent field, malformed
  JSON, wrong JSON type, empty body) all return the identical
  `400 {"error":"title is required"}` — a client cannot tell a parse failure
  from a validation failure; non-numeric, negative and overflowing `{id}` all
  land in the 404 partition; `text/plain` is accepted (no 415); 201 carries no
  `Location`; 405 *is* emitted with a correct per-path `Allow` (`GET, HEAD` /
  `POST`) and nothing tests it; a whitespace-only title is accepted though AC2
  says "missing or empty"; a duplicate POST silently creates a second task.
- **The reviewer-direction rule was executed** (`/tmp/tdt-check/derive.py`):
  the pipeline's 19 derived items for PROJ-101, classified AC-stated vs
  derived, matched against both fixture branches. Result — the AC-stated rule
  reproduces the item-2 ground-truth verdict vector **exactly on both
  branches** (full: covered×4; partial: covered/missing/partial/missing, with
  AC3's `partial` falling out of the unasserted `completed` flag), and the
  derived rule adds 9 gaps + 1 recorded N/A **without moving any verdict**.
  8 of those 9 gaps are behaviours the live probe above confirmed actually
  exist and are untested; only the BVA title-length boundary is unconfirmable
  (the in-memory fixture has no storage limit). Every one of the 19 items
  classified AC-stated-or-derived without a coin flip.

Deliberately **not** run: the skills themselves as live agents. The item-4
smoke eval is the clean-room exercise, and PROVING-GROUND rule 1 forbids the
session that wrote the reference from grading it. Note also that the derived
item list here was hand-authored by the builder from the pipeline — what the
script *executes* is the classification and matching rule, not the derivation
itself. Rung 1 is where a fresh session's derivation gets compared to a
reference one.

**Learned / for the next iterations.**

- The identical 9-gap list on both branches is the single most useful number
  this item produced: it is direct evidence for keeping the gap channel and
  the verdict channel separate, and it suggests a scorecard metric for rungs
  1–5 — *gap-list stability* (should be near-identical across branches of one
  service) versus *verdict accuracy* (must discriminate). If a future run's
  gap list starts varying by branch, the reviewer is deriving from the code
  under review instead of from the ACs.
- Item 4 now has a second thing to check besides verdicts: whether a
  clean-room reviewer reports the derived gaps at all. A skill that produces
  the right verdicts and an empty Gaps section has silently dropped step 7.
- PDF-sourced citations need machine checking, full stop. Two of my quotes
  were wrong in ways that read perfectly fluently — one dropped parenthetical
  and one inverted example. Any rubric line citing a standard should be
  greppable against a downloaded copy of that standard as part of verify.

**For planning sync:** this closes the "where do edge cases come from?"
question with a citable answer instead of model intuition — ISTQB for the
model-based half, Hendrickson for the experience-based half, RFC 9110 for the
protocol half — and, more importantly for D1, it separates *coverage gaps*
from *AC verdicts* as two independent output channels. The gating carve-out
sketched in item 2 ("gate only on high-confidence verdicts") stays coherent
under this: gaps inform, verdicts decide, and neither leaks into the other.
