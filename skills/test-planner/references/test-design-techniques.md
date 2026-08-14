# Test-design techniques

How to derive the cases an acceptance criterion implies but does not say.
Read by **both** skills, in opposite directions:

- **test-planner Step 4** runs the derivation *forward*: AC → coverage items
  → planned tests (mostly `Type: edge` / `Type: failure`, often
  `Verifies: implied`).
- **test-reviewer Step 3** runs it *backward*: the same derivation over the
  same ACs, then asks which derived items nothing in the MR exercises. Those
  are gap lines, not verdicts — see **Reviewer direction: gaps, not
  verdicts** at the bottom; that rule is load-bearing.

Both directions run the **same pipeline in the same order**, because that is
what makes "the planner missed it" and "the reviewer found a gap" comparable
statements about one list rather than two people's intuitions.

Sources: black-box techniques and their coverage criteria are ISTQB CTFL
v4.0.1 (2024-09-15) §4.2 and §4.4.3; the heuristic sweep is the Test
Heuristics Cheat Sheet (Hendrickson / Lyndsay / Emery, © 2006 Quality Tree
Software, maintained edition © 2022 Ministry of Testing); HTTP failure-path
semantics are RFC 9110 (June 2022). Everything marked *house* is ours to
maintain — no source claimed.

## The pipeline (fixed order, do not reorder)

For **each AC**, in this order. Order is load-bearing: BVA operates on the
partitions EP produced, and the heuristic sweep is deliberately last so it
only has to catch what the models cannot reach.

| # | Step | Trigger — run it when… | Produces |
|---|------|------------------------|----------|
| 1 | Variables & outcome | always | the AC's input variables and its one observable outcome |
| 2 | Equivalence partitioning | always, per variable | valid + **invalid** partitions |
| 3 | Boundary value analysis | a partition is *ordered* | boundary values (+ neighbors) |
| 4 | Decision table | ≥2 conditions jointly decide the outcome | feasible rule columns |
| 5 | State transition | the AC names a status, lifecycle or "only when X" | valid + invalid transitions |
| 6 | Heuristic sweep | always | what steps 2–5 structurally cannot see |
| 7 | Protocol failure paths | the AC touches an HTTP endpoint / consumer / write path | status-code and header obligations |

Steps 2–5 are *model-based*: they enumerate from a model of the input space,
so two runs over one AC produce the same items. Step 6 is *experience-based*
and does not — which is exactly why it is a bounded checklist here and not an
invitation to improvise (§4.4.3 warns that checklist items which are too
general are worthless; ours name a concrete attack or they are cut).

**Coverage items are not tests.** ISTQB counts coverage items; the plan
counts tests, and one test routinely covers several items (a transition
sequence, a table-test row set). Collapse before you write — see
[Budget](#budget-when-to-stop).

**The derivation adds no syntax to the plan.** There is no `Technique:`
field: `plans/<KEY>.md` stays byte-compatible with the test-planner CLI
([plan-format.md](plan-format.md)). Attribution, where it helps a reader,
goes in the test *title* ("Title at max length is accepted") — never in a new
field.

## 1. Variables and outcome

Name, per AC: every input variable it constrains (body field, path/query
param, header, message field, and the *pre-existing state* the AC assumes),
and the single observable outcome it promises. An AC with two outcomes is two
ACs' worth of tests; an AC with no observable outcome is a Step 3 triage
failure, not a derivation input — flag it and stop.

State counts as an input. "GET /tasks/{id} for an existing task" has two
variables: the id *and* whether a task with that id exists.

## 2. Equivalence partitioning (ISTQB §4.2.1)

> Equivalence Partitioning (EP) divides data into partitions (known as
> equivalence partitions) based on the expectation that all the elements of a
> given partition are to be processed in the same way by the test object.
> [ISTQB CTFL v4.0.1 §4.2.1]

Partition every variable into **valid** and **invalid** partitions. Coverage
criterion: at least one test case per identified partition, *including the
invalid ones* — that clause is where most plans lose. Coverage = partitions
exercised / partitions identified [§4.2.1].

With several variables, the criterion is **Each Choice** — every partition of
every set exercised at least once, combinations not considered [§4.2.1,
citing Ammann 2016]. Each Choice is our default; full Cartesian combination
is never the default (see Budget).

Worked, on the fixture tasks API (`POST /tasks {"title": string}`):

| variable | valid partitions | invalid partitions |
|---|---|---|
| `title` | non-empty string | absent field · empty string · non-string type (`{"title": 42}`) |
| request body | well-formed JSON object | malformed JSON · empty body · non-object JSON |
| `{id}` (GET) | numeric id of an existing task | numeric id of no task · non-numeric · negative · beyond int range |

Two findings fall straight out, and both are the *planner's* to raise before
code exists: the API answers `malformed JSON` and `empty title` with the same
400 + `"title is required"` (two partitions, one response — a client cannot
tell a parse failure from a validation failure), and a non-numeric `{id}`
lands in the `not found` partition rather than a bad-request one. Neither is
visible from the AC text; both are visible from the partition table.

## 3. Boundary value analysis (ISTQB §4.2.2)

BVA exercises the boundaries of *ordered* partitions only — length, count,
amount, timestamp, page size. An unordered partition (an enum, a type) has no
boundary and gets no BVA line; say so rather than inventing one.

- **2-value BVA**: the boundary value and its closest neighbor in the
  adjacent partition [§4.2.2, citing Craig 2002 / Myers 2011].
- **3-value BVA**: the boundary and *both* neighbors [§4.2.2, citing Koomen
  and O'Regan]. More rigorous, and the syllabus's own example shows why: for
  `if (x ≤ 10)` mis-implemented as `if (x = 10)`, neither 2-value datum
  (x=10, x=11) detects the defect — x=9, which only 3-value derives, does.

*House rule:* **default to 2-value; use 3-value when the AC states the
boundary itself** ("titles up to 200 characters"), because that is precisely
where an off-by-one is both plausible and contractual. Record which variant
you used — a reviewer checking boundary coverage needs to know which items
were on the list.

Boundaries our stack forgets, in order of how often they bite: string length
at the storage limit (Spanner `STRING(n)` truncation is silent in some client
paths), collection size 0 / 1 / max page, numeric 0 and negative, and the
timestamp boundary (an event and a row written either side of a second/day
boundary).

## 4. Decision tables (ISTQB §4.2.3)

When ≥2 conditions jointly decide the outcome, tabulate: conditions as rows,
combinations as columns, actions below. Delete infeasible columns.

> In decision table testing, the coverage items are the columns containing
> feasible combinations of conditions. To achieve 100% coverage with this
> technique, test cases must exercise all these columns.
> [ISTQB CTFL v4.0.1 §4.2.3]

`POST /tasks` has two conditions and three feasible columns:

| | R1 | R2 | R3 |
|---|---|---|---|
| body parses as JSON | T | T | F |
| `title` non-empty | T | F | – |
| → 201 + created task | ✓ | | |
| → 400 + error | | ✓ | ✓ |

Three columns, and the AC set names two of them (AC1 = R1, AC2 = R2). R3 is
the column nobody wrote down: it is a real, reachable behavior with no AC, so
it becomes a `Verifies: implied` test — and, per §2 above, a note that R2 and
R3 are indistinguishable to the client.

The table's other job is **finding gaps and contradictions in the
requirements** [§4.2.3]: a column two ACs answer differently is a
contradiction to raise under **AC issues**, not a case to plan around. (Rung 5
of the proving ground is built out of exactly this.)

## 5. State-transition testing (ISTQB §4.2.4)

Trigger: the AC mentions a status, a lifecycle, or a precondition of the form
"only a *pending* order can be paid". Build the **state table** — rows =
states, columns = events — because unlike a state diagram it shows the
invalid transitions explicitly, as empty cells [§4.2.4].

Coverage criteria, all three defined in §4.2.4:

- **all states** — every state reached. Weakest; achievable without
  exercising all transitions.
- **valid transitions (0-switch)** — every valid transition exercised. The
  most widely used criterion, and *our default*.
- **all transitions** — every cell of the state table, i.e. all valid
  transitions plus an *attempt* at every invalid one. Guarantees the other
  two. §4.2.4 recommends it as a minimum for mission/safety-critical
  software.

*House rule:* **all-transitions is the default for any state machine exposed
over HTTP.** An invalid transition on a public API is not an exotic path — it
is what a retried request, a double-click or a competing consumer produces,
and the response to it (409 vs 200-and-corrupt) is a contract. §4.2.4's own
caution applies: exercise **one invalid transition per test case**, or a
defect in the first masks the second.

Worked, on the fixture orders API (states `pending`/`paid`/`cancelled`,
events `pay`/`cancel`):

| state \ event | pay | cancel |
|---|---|---|
| pending | → paid (200) | → cancelled (200) |
| paid | *invalid* — 409 `invalid_transition` | *invalid* — 409 |
| cancelled | *invalid* — 409 | *invalid* — 409 |

Counts, which is the point of writing the table: 3 states, **2** valid
transitions, **6** all-transition items. A suite that pays and cancels a
pending order has 100% valid-transitions coverage and 33% all-transitions
coverage — and a plan that stops there has left the entire 409 contract
untested while looking complete.

Off the table but on the list for a distributed stack: the **same** event
delivered twice (idempotence — see Kafka checklist), and a transition
attempted concurrently from two callers.

## 6. Heuristic sweep (Hendrickson cheat sheet)

Experience-based, ISTQB §4.4.3 checklist-based testing. The full cheat sheet
covers surfaces we do not have (browser, mobile, a11y). Pruned to what can
fail in a Go HTTP + Spanner + Kafka service — this table *is* the checklist;
walk it once per ticket, not once per AC:

| Heuristic | The attack, in our stack |
|---|---|
| **Data type attacks — strings** | very long, empty, whitespace-only, leading/trailing space, Unicode + emoji (grapheme vs byte length), accented and CJK characters, delimiters and quotes, embedded newline, SQL/injection payloads through a parameterized path |
| **Data type attacks — numbers** | 0, negative, `MaxInt64`, overflow of the parse (`strconv.Atoi`), float where int expected, scientific notation, leading zeros |
| **Data type attacks — time/date** | timezone ≠ UTC, DST edges, leap day, epoch 0, far-future, clock skew between writer and reader, timeout expiry |
| **Count (0 / 1 / many)** | empty collection, exactly one, page-size+1; *cardinality* after an upsert or a replayed event |
| **Goldilocks (too big / too small / just right)** | body at and over the size limit, batch at and over the max |
| **CRUD / Follow the data** | create → read → update → delete the same entity end-to-end, asserting each stage's external contract (row, event, response) |
| **Sequences** | operations out of order, repeated, reversed; the retry after a partial failure |
| **Interruptions** | client disconnect mid-request, context cancellation, dependency timeout, redeploy mid-flow |
| **Multi-user / flood** | two callers mutating one entity; concurrent consumers on one key |
| **Dependencies** | the "has-a" chain — parent missing, parent deleted while child in flight |
| **Constraints** | required field violated, unique constraint violated (the second insert is the test) |
| **Input method** | the same behavior reached via HTTP and via a consumed event — do both paths enforce the same rules? |
| **TouchPoints** | every interface in and out: HTTP, Kafka in, Kafka out, Spanner, config/flags |
| **State analysis** | feeds step 5 — enumerate states/events before assuming there is no state machine |

Out of scope for this plugin's domain and recorded **once** as N/A rather than
per-AC: web/browser navigation, preferences, accessibility, mobile/device.
Naming them is not ceremony — a report that silently omits a category reads
identically to one that considered and cleared it.

*House rule:* the sweep may only *add* cases. It never downgrades or
overrides a model-derived case from steps 2–5.

## 7. HTTP failure paths (RFC 9110)

The rubric's **HTTP endpoint** checklist in
[completeness-rubric.md](../../test-reviewer/references/completeness-rubric.md)
says *which* paths we require — that selection is house standard. RFC 9110
fixes *what each one means* and, in two cases, what the response **MUST**
contain. Assert the meaning, not just the number.

| Path | Code | RFC 9110 | What the test must assert |
|---|---|---|---|
| malformed syntax / framing | 400 | §15.5.1 | server "cannot or will not process… due to something perceived to be a client error" |
| well-formed but semantically invalid | 422 | §15.5.21 | content type understood, syntax correct, instructions unprocessable — *distinct from 400 and from 415* |
| unsupported media type | 415 | §15.5.16 | rejection is due to the format, not the values |
| no/invalid credentials | 401 | §15.5.2 | **MUST** carry a `WWW-Authenticate` header with a challenge — assert the header, not only the status |
| authenticated but refused | 403 | §15.5.4 | refusal understood-but-forbidden; note §15.5.4 permits answering 404 instead to hide existence — if the API does that, the *plan* must say so, or the 404 test is asserting a lie |
| no current representation | 404 | §15.5.5 | temporary-or-permanent is undefined; 410 (§15.5.11) is preferred when known permanent |
| method known, unsupported here | 405 | §15.5.6 | **MUST** generate an `Allow` header listing supported methods — assert its contents |
| conflict with current state | 409 | §15.5.10 | conflict with the *current state of the target resource*, resolvable and resubmittable — the state-machine code (step 5) |
| precondition evaluated false | 412 | §15.5.13, §13.1 | conditional-request guard (`If-Match`) actually enforced |
| content too large | 413 | §15.5.14 | rejected before processing; `Retry-After` if temporary |
| resource created | 201 | §15.3.2 | the created resource is identified by `Location` **or** the target URI — if the API sets neither, that is a finding |
| temporary overload / maintenance | 503 | §15.6.4, §10.2.3 | mapped, not a hang; `Retry-After` where the API claims it |
| upstream did not answer in time | 504 | §15.6.5 | dependency timeout maps to a defined response, no hang, no partial side effects |

**Idempotency** [§9.2.2]: "the intended effect on the server of multiple
identical requests… is the same as the effect for a single such request", and
PUT, DELETE and the safe methods **are** idempotent. So a PUT or DELETE
endpoint owes an idempotency test, and the assertion is *cardinality* — one
row, one event — not a second 200. §9.2.2 exists because clients retry
automatically after a connection failure, which is also why the Kafka
duplicate-delivery item next door is the same test wearing a different hat.

**POST is not idempotent** by §9.2.2, so a create endpoint owes instead a
*defined* duplicate-submission behavior (second create, or 409, or an
idempotency key) — and the plan must state which, since all three are
legitimate and only one is implemented.

For Kafka consumers and Spanner writes, the failure-path lists in the rubric
stand as they are: house standard, no RFC to defer to.

## Budget: when to stop

Derivation must terminate, and a plan nobody reads is a plan nobody follows.

- **Collapse to the layer.** A 12-row string-attack matrix is **one** planned
  unit test with a table of cases — not 12 plan entries. Integration gets
  **one representative per partition / boundary / status code**, chosen for
  the highest-consequence value.
- **Each Choice, not Cartesian.** Combine partitions across variables only up
  to Each Choice coverage [§4.2.1] unless a specific risk argues for a
  decision table — and then it is a decision table, with its columns written
  down.
- **All-transitions, one invalid per test** [§4.2.4].
- **The sweep is a walk, not a search.** One pass over the table in §6 per
  ticket. An entry with no plausible failure in this ticket is N/A with a
  word of why.
- **Stop rule:** every partition, boundary, feasible column and transition
  cell is either mapped to a planned test id or written down as a deliberate
  omission. Nothing derived may simply vanish — undocumented pruning is how a
  derived set becomes uncheckable by the reviewer.

## Reviewer direction: gaps, not verdicts

Re-derive from the ACs (never from the MR's tests — deriving from the code
under review just re-describes what was built), then diff against the
evidence inventory. What you do with a miss:

1. **The AC itself names the item** — the AC says "titles up to 200
   characters" and no test touches 200/201, or the AC names the 409 and the
   invalid transition is untested. The item *is* the AC's observable outcome:
   the verdict is `partial`, and the missing axis is named as usual.
2. **The item is derived, not stated** — an unexercised invalid partition, an
   untested boundary the AC never mentioned, a 405 path. The AC's own verdict
   is **unaffected**. It goes under **Gaps and recommended tests** with the
   technique named ("BVA, `title` length: no case at the storage limit") and
   a ready-to-add skeleton.

Rule 2 is the load-bearing one. Letting derived items downgrade AC verdicts
would make every review INSUFFICIENT — the derived set is always larger than
any AC set, so the verdict column would stop discriminating and start meaning
"the reviewer ran its checklist". The rubric already places uncovered
applicable failure paths as gaps "even when every AC is green"; this section
is that rule generalized to all seven steps.

Confidence is unaffected either way: gaps are about coverage, confidence is
about how sure we are of a verdict. Do not let a long gap list talk you into
lowering a verdict's confidence.

## Sources

- **ISTQB Certified Tester Foundation Level Syllabus v4.0.1** (2024-09-15),
  §4.2.1 equivalence partitioning, §4.2.2 boundary value analysis, §4.2.3
  decision table testing, §4.2.4 state transition testing, §4.4.3
  checklist-based testing. Quoted definitions and coverage criteria are the
  syllabus's own wording.
- **Test Heuristics Cheat Sheet** — Elisabeth Hendrickson, James Lyndsay,
  Dale Emery; © 2006 Quality Tree Software, Inc.; maintained edition © 2022
  Ministry of Testing Ltd (ministryoftesting.com). §6's table is a pruned
  restatement for this domain, not the full sheet.
- **RFC 9110 — HTTP Semantics** (Fielding, Nottingham, Reschke; STD 97, June
  2022). Section numbers as cited; `MUST` obligations quoted.
- **House standard** (no external source claimed): the pipeline and its
  order, the 2-value/3-value default, all-transitions-by-default for HTTP
  state machines, the pruned heuristic table, which failure paths we require,
  the budget rules, and the gaps-not-verdicts rule.
