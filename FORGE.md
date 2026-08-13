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
- [ ] 2. **Confidence in the report.** Per-AC confidence column
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
- [ ] 5. **Golden eval repo** (~/forge/test-skills-eval): small Go API
      slice (order-lifecycle style; Spanner emulator + Kafka via
      testcontainers if docker cooperates, in-memory fakes if not — note
      which in the Log), godog via go test+TestingT, a plan file, and
      branches with SEEDED, DOCUMENTED coverage states: full; one AC
      missing; layer mismatch; unwired scenario; name-gaming test (right
      name, no assertion); missing dual-write half. EXPECTED.md per branch.
- [ ] 6. **Eval sweep + calibration.** Run the reviewer against every
      golden branch 3×. Measure accuracy (verdict vs EXPECTED.md) and
      stability (same verdict across runs). Log the matrix. Where wrong or
      unstable: tighten rubric wording (rules, not vibes), re-sweep.
- [ ] 7. **Planner eval.** Run the planner against 2-3 synthetic tickets
      (write them, with known-good reference plans): does it catch the
      untestable AC? produce the implied failure paths? reuse step
      vocabulary? Log + refine.
- [ ] 8. **Polish pass from evidence.** Fold everything the evals taught
      back into: rubric, godog-standards, SKILL.md steps, README. Write the
      planning-sync summary in the Log (eval design + results are research
      question D1 evidence).

## Log

*(one dated entry per iteration: what was done, what was learned, what
changed in the queue)*
