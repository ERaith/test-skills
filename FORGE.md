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

- [ ] 1. **Agent fan-out in both skills.** Planner: parallel context agents
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
