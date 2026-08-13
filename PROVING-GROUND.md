# Proving ground: staged real-world evals for the test agents

The planner and reviewer earn confidence on an escalating ladder of
realistic scenarios before team rollout. Each rung adds one real-world
complication; the agents must pass the rung gate before Forge builds the
next. Output doubles as the Phase 2→3 graduation evidence (research
question D1).

Location: `~/forge/test-skills-eval` — local git only, branches per
coverage state. Do NOT create a GitHub repo for it (eraith decides later).

## Per-rung structure

```
specs/RUNG<n>-<name>.md        Confluence-style spec for the rung services
tickets/<KEY>.md               simulated Jira tickets — MIXED quality (below)
plans/reference/<KEY>.md       hand-authored reference plan = planner ground truth
EXPECTED/<branch>.md           per-branch reviewer ground truth
```

Ticket file format (planner consumes these as pasted content — this box has
no Jira MCP, which conveniently exercises the fallback path):

```markdown
# <KEY>: <summary>
Epic: <epic name> · Links: <related KEYs>
## Description
## Acceptance criteria
- ...
```

**Quality mix per rung (mandatory):** one crisp ticket; one with a
vague/untestable AC (the planner MUST flag it, not plan around it); one
whose ACs omit obvious failure paths (planner must add `Verifies: implied`
tests). Rung 5 adds contradictory ACs and ticket-vs-MR scope mismatches.

**Ground-truth discipline:** EXPECTED/ and reference plans are written from
the spec+tickets BEFORE any agent runs — never derived from agent output.
No grading your own homework.

## The ladder

- **Rung 0** — CLI fixtures baseline (FORGE.md item 4). Sanity only.
- **Rung 1 — single HTTP service.** In-memory store, godog via go test +
  TestSuite{TestingT}. Branches: full / one-AC-missing / gamed-test (right
  name, no assertion) / unwired-scenario.
- **Rung 2 — + Spanner emulator.** Persistence tickets: back-door row
  assertion steps, update-vs-insert cardinality. New branches:
  missing-cardinality-check / symmetric-roundtrip-trap (write+read both
  wrong, only back-door catches it).
- **Rung 3 — + Kafka (Redpanda testcontainer).** Producer+consumer tickets,
  dual-write. New branches: missing-dual-half / no-poison-test /
  non-idempotent-consumer.
- **Rung 4 — 3 microservices.** Orchestrator → rules-engine →
  persist+publish (mirrors the production copper/CRE/CPVA shape). Contract
  seams, peers faked per component-test discipline, one cross-service
  ticket. Reviewer must assemble evidence across service dirs.
- **Rung 5 — the mess batch.** Contradictory ACs; MR implements more than
  the ticket (untracked work) and less (silent scope cut); plan written,
  then implementation drifted; ticket edited after the plan. The reviewer
  report must SAY these things, not silently pick a side.

## Scoring (append per-rung scorecard to FORGE.md Log)

- **Planner vs reference plan:** AC coverage complete? every seeded vague
  AC flagged? implied failure paths present? layer assignment correct?
- **Reviewer vs EXPECTED:** per-AC verdict exact-match %; hallucinated
  citations (MUST be 0); SUFFICIENT/INSUFFICIENT call correct; stability
  across 3 runs (same verdicts).
- **Advance gate:** ≥90% per-AC accuracy, 0 hallucinated citations, 3/3
  stable, all vague ACs flagged. Below gate → tighten rubric/skills (rules,
  not vibes), re-run the SAME rung. Log every calibration change.

Infra note: docker is available; if testcontainers misbehave on this box,
fall back to in-process fakes and mark the rung log accordingly — do not
silently skip the infra dimension.
