# test-skills

Claude Code plugin for the SDET workflow on Go API services with Spanner
persistence and Kafka messaging. Two skills:

- **`/test-skills:test-planner TICKET-KEY`** — run right after ticket
  breakdown. Gathers the Jira ticket (+ linked Confluence spec) and the
  target repo's existing godog suite, triages acceptance criteria for
  testability, and writes a per-AC test plan (`plans/<KEY>.md`) with layer
  assignments and draft Gherkin that reuses the repo's step vocabulary.
- **`/test-skills:test-reviewer MR-REF [TICKET-KEY]`** — run when an MR is
  up. Classifies every AC and planned test as covered / partial / missing
  with grep-verified evidence from Go unit tests AND godog feature files,
  walks the failure-path checklists (HTTP / Kafka consumer / Spanner
  persistence), and reports SUFFICIENT / INSUFFICIENT. Evidence is
  inventoried by parallel agents and every `covered` verdict is then
  attacked by an independent skeptic agent — only verdicts surviving
  refutation stay `covered`. Every verdict carries
  a derived confidence (`high`/`medium`/`low`) and a line saying what would
  raise it, so a green report tells you where it is thin. **Advisory only**
  — it does not gate; an SDET makes the merge call.

Both skills are explicit-invocation only (`disable-model-invocation`), so
they never fire as a side effect of unrelated conversation.

## Install (team)

```
/plugin marketplace add ERaith/test-skills
/plugin install test-skills@test-skills-marketplace
```

## Prerequisites

- **Jira MCP** connected in your Claude Code session (the planner and
  reviewer fetch tickets through whatever Jira tools the session exposes;
  they fall back to pasted ticket content).
- **`glab`** authenticated against our GitLab (MR diffs/notes, repo reads).
- For running `@integration` suites during review: a container runtime
  (Docker, or podman with `DOCKER_HOST` set).

## Shared standards (the interesting part)

The skills are thin; the standards they apply live in versioned reference
files — change them by MR, and both skills pick the change up:

- `skills/test-planner/references/plan-format.md` — the canonical plan
  format, byte-compatible with the `test-planner` CLI.
- `skills/test-planner/references/agent-fanout.md` — the agent fan-out
  contract used by both skills: the planner's context lanes (ticket graph /
  spec / repo suite inventory), the reviewer's evidence lanes (unit tests /
  feature files / step wiring — deliberately blind to each other) and its
  per-verdict skeptic agents, plus the return envelope, the merge rules
  (agents gather, the caller decides; every agent citation re-grepped) and
  the no-subagent fallback.
- `skills/test-planner/references/gobdd-standards.md` — the team's internal
  `gobdd` framework (a godog wrapper): the `gobdd.Run` runner, step **packs**
  with `@step:` annotation catalogs, built-in `api`/`kafka`/`database`/…
  packs (steps wired by import), the fixed `features/` layout and per-PIB job
  configs, legacy tag syntax, and the request-response Kafka + dual-write
  (message + SQL row) shapes.
- `skills/test-reviewer/references/completeness-rubric.md` — verdict
  definitions, the evidence bar (cite-or-missing, wired-means-wired), the
  confidence derivation (structural: skeptic-survived + executed = `high`;
  static-only caps at `medium`), assertion-quality smells (testsmells.org),
  layer matching, and the failure-path checklists.

## Relationship to the `test-planner` CLI

Same plan format, same verdict vocabulary (`covered`/`partial`/`missing`),
same confidence buckets (`high`/`medium`/`low`, per verdict — so reports from
either side are comparable verdict-for-verdict), same exit-code philosophy.
Where the CLI asks the model how confident it feels, the skills derive
confidence from structure (was it executed? did it survive refutation?). The
CLI is the deterministic one-shot for CI;
these skills are the interactive/agentic surface for humans working a
ticket. Plans produced by either are readable by both.

## Releasing changes

Just merge to `main`. The plugin intentionally has **no version field**:
without one, Claude Code tracks the git commit, so teammates pick up rubric
and skill changes on their next plugin update — no semver bump to forget
(a pinned version that nobody bumps means nobody gets updates).

## Roadmap

Phase 1 (now): advisory skills, SDET approval required on MRs.
Phase 2: reviewer runs automatically per-MR, posts its report, SDET still
gates. Phase 3: agent verdict gates, with carve-outs (auth, contracts,
schema) and sampled human audits. Graduation is evidence-based — see the
planning docs in the team workspace.
