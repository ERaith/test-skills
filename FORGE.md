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

## Queue

- [x] 1. **Agent fan-out in both skills.** Planner: parallel context agents
      (ticket graph incl. dependent/blocking tickets + epic; Confluence
      spec; repo suite inventory). Reviewer: parallel evidence-inventory
      agents (unit tests / feature files / step wiring), then a skeptic
      agent per `covered` verdict prompted to REFUTE it — only verdicts
      surviving refutation stay `covered`.
- [x] 2. **Confidence in the report.** Per-AC confidence column
      (high/medium/low) + "what would raise it" line. Confidence is
      structural: skeptic-survived + executed = high; static-only evidence
      caps at medium; ad-hoc plan caps overall confidence. Mirror the CLI
      fixtures' per-verdict confidence semantics.
- [ ] 3. **test-design-techniques.md reference** (planner references/, read
      by both skills): equivalence partitioning, boundary value analysis,
      state-transition testing, decision tables (ISTQB-grounded);
      Hendrickson test-heuristics categories; RFC 9110 grounding for the
      HTTP failure-path checklist. Wire into planner Step 4 (edge-case
      derivation per AC) and reviewer Step 3 (edge-case gap check).
- [ ] 4. **Smoke eval vs CLI fixtures.** Run the reviewer skill logic
      against ~/forge/test-planner fixtures: tickets/PROJ-101.json +
      features-full (expect SUFFICIENT) and features-partial (expect
      INSUFFICIENT). Compare against the CLI's own plans/ and reports if
      present in the fixtures. Record agreement + disagreements in the
      Log; fix rubric/skill where the skill is wrong; note CLI
      discrepancies (do not modify the CLI copy).
- [ ] 5. **Proving ground rung 1** (see PROVING-GROUND.md): build the
      single-service scenario + mixed-quality tickets + reference plans +
      EXPECTED branches, then run the full eval cycle (planner + reviewer
      3x, scorecard, calibrate until the advance gate passes).
- [ ] 6. **Proving ground rungs 2-3**: add Spanner emulator rung, eval
      cycle to gate; then Kafka rung, eval cycle to gate. One rung at a
      time — never build ahead of a failed gate.
- [ ] 7. **Proving ground rungs 4-5**: 3-microservice rung, eval cycle;
      then the mess batch (contradictory ACs, ticket-vs-MR scope
      mismatches). Scorecards to the Log.
- [ ] 8. **Polish pass from evidence.** Fold everything the evals taught
      back into: rubric, godog-standards, SKILL.md steps, README. Write the
      planning-sync summary in the Log (eval design + results are research
      question D1 evidence).

## Log

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
