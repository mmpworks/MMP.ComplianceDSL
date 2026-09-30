# RT-2: Language design, compiler engineering, cost and scope

> Note on publication: this review cites "Research 01/02", internal research notes that are not published. The drift items it cites are in `docs/design/drift-register.md`. The v0 design it reviews is summarized in `docs/design/decisions.md`.

Reviewer lens: senior staff engineer who has watched internal DSLs fail. Hostile by brief.
Reviewed: `docs/design/00`–`03`, `decisions.md`, `docs/research/01`–`02`, checked against
`smuchow1962/ffiec`, `ffiec-chain-of-custody/spec/test-vectors`, `formbuilderdsl`,
`herald.tesseraseal.vectorcompiler`. Date: 2026-09-30.

## Verdict

v0 correctly diagnoses the disease (18 drift items, four hand copies of every table) and
prescribes a cure roughly forty times larger than it. CDL is not one language: it is seven
(constant/enum tables, a typed expression language, a state-machine notation, a relational
calculus with a query planner, an imperative step procedure with modes and flags, a
reason-template formatter, and a mutation language for test vectors), each needing a
normative semantics written once and implemented twice, by one maintainer, through agents
who share one model's misreadings. Worse, the design's central safety argument inverts
itself: a shared rule pack makes Go, Python and .NET **one witness of the spec with three
interpreters**, which is weaker evidence under `GOVERNANCE.md:42` than the independent hand
verifiers it replaces. The design's own worked §10.84 example already mistranscribes the Go
code it claims to transcribe in four ways, which is the common-mode failure the charter
lists as risk 1. The 80% value (kill drift of constants, enums, reasons, exit codes,
transition tables, field schemas) needs no grammar at all: a closed JSON data file, a
~1k-line Go validator/generator, small hand-written table engines per language, and
hand-written procedural rules where independence actually matters. Build that. Revisit a
textual language only if a counted backlog of genuine expressions justifies it after P2.

## Findings

### F1. A shared rule pack defeats the independence it claims to protect. Severity: Critical

**Attacked:** 00 R1, 01 §1 ("share data, not code"), 03 §5 risk 1, 03 P5.

The witness requirement (C2) exists so that an error in one reading of the spec is caught by
another reading. v0 puts the *reading* (the transcription in `rules/`) in one place and
makes only the *interpreters* independent. Two runtimes agreeing on a wrong rule is a green
differential. The only independent check on transcription left is the author-declared
expected outcome (DSL-007), written by the same author, often in the same agent session,
as the rule. P5 then moves Herald.Compliance's validators into the pack, so the planned
"three independent conforming implementations" become three engines over one
transcription. A foundation reviewer will count that as one implementation. Note also the
circularity: the second runtime mostly exists to catch bugs in the first runtime, which
exists only because there is an interpreter.

**Fix.** Share *tables* (closed data with one answer per question, R10) and keep
*procedural logic* hand-written per witness. Tables are low-risk to share because they are
checked against the spec by quote and by generated Markdown review; procedures are where
independent readings earn their keep.

### F2. The design's own §10.84 example is a mistranscription. Severity: Critical

**Attacked:** 02 §6; charter "every consumer stays an independent witness".

Compared with `verifier/internal/verify/communication_preapproval.go`:

1. Go returns no anomaly when no send exists (`checkOneCommunication`, "a communication
   approved but not yet sent is a normal pending state"). CDL's
   `require exists(approval)` fires for every unsent retail communication.
2. Go reports the **send's** seq (`sendSeq`); CDL renders `{c.seq}`, the communication's.
3. Go skips review events with no `signed_at_utc`; CDL has no such filter.
4. Go takes the first match; CDL's `one` is undefined for 0 or >1 matches (see F5).
5. With no send, `approval.signed_at_utc <= send.applied_at_utc` is `undefined`, and the
   stated §10.84 policy "undefined is a violation" turns it into an anomaly. Go does the
   opposite.
6. Paths: `r.audit.review.role` cannot say whether it means the flat key
   `"audit.review.role"` (what Go decodes) or a nested object. That is drift item D17,
   silently re-created by the path syntax.

Similarly, the §5 machine template renders `{prev.to}` unquoted where vector 037 pins
Python `!r` quoting, so R9's byte oracle fails on the first machine vector, and the
`opened -> opened` self-loop quietly decides D13 inside an example. An author who
transcribed this carefully in a design doc produced six divergences in 18 lines; the pack
would propagate every one to all runtimes at once.

**Fix.** Treat this as the empirical result for F1. Relations stay hand-written per
language until they are pinned by negative vectors that cover each branch above.

### F3. DSL-003's dismissals are partly inaccurate and miss the real comparison. Severity: High

**Attacked:** decisions.md DSL-003.

Checked 2026-09-30 with `go mod tidy` + `go list -deps`:

- **cel-go dependencies: accurate.** `cel.dev/cel-go/cel` pulls `antlr4-go/antlr`,
  `google.golang.org/protobuf`, `genproto`, `cel.dev/expr`, `golang.org/x/exp`,
  `golang.org/x/text` and `go.yaml.in/yaml/v3`. It breaks C1.
- **"No Python CEL independent of Google's": inaccurate.** `cel-python` (Cloud Custodian,
  pure Python over Lark) is independently written, and CEL ships a public conformance suite,
  which is exactly the runtime corpus v0 proposes to hand-build.
- **Starlark: not considered at all.** `go.starlark.net/starlark` is pure Go but *not*
  stdlib-only: `go list -deps` shows `golang.org/x/sys/unix`. It has no recursion and no
  unbounded loops by default, is deterministic and hermetic (R6, R7 nearly for free), and
  has independent Go, Java and Rust implementations. Strict C1 rejects it on one
  `x/sys` import; the decision log should say so rather than omit it.
- **JSON Schema "draft drift": a straw man.** Nobody needs a general validator. Go
  `encoding/json` with `DisallowUnknownFields` and Python `pydantic` with `extra="forbid"`
  (already a C6 dependency) validate a closed shape with zero third-party risk.
- **The missing comparison is option (a): typed data plus hand code.** DSL-003 compares a
  custom language against general policy engines and wins easily. It never compares against
  "JSON tables plus hand-written procedures", which is the cheapest credible design and the
  one that best satisfies C2.

**Fix.** Rewrite DSL-003 with the four options (custom language, embedded DSL, CEL/Starlark
for expressions only, data + hand code), with measured dependency lists, and pick (a).

### F4. CDL is seven languages, each needing its own normative semantics. Severity: High

**Attacked:** 00 Goal 1, 02 §1 "a small declarative language", 03 §5 risk 2.

Count the semantic domains: tables; total expressions with three-valued logic; state
machines with ordered checks and party keys; relations with `for/one/all/exists`, `let`,
indexes and a cost-based planner; an imperative procedure with `let` scoped across steps,
`for` loops, short-circuit, `after` ordering, modes and flags; a template formatter with
typed slots and format directives; and a vector language that builds payloads, mutates at
three layers and re-chains with a MAC or attacker key. Each needs a section of
`cdl-language.md`, a lowering, an IR node set, a Go evaluator, a Python evaluator and
conformance cases. FormBuilderDSL, one grammar with one runtime and no type system, is 7.3k
source lines, 12.4k test lines and 26 regression rounds. CDL is several times that surface
with twice the runtimes.

**Fix.** Keep only the domains that are tables. Everything with control flow stays in host
code.

### F5. Semantics in 02 are underspecified where two runtimes will diverge. Severity: High

**Attacked:** 02 §4–§9, 01 §4.

Each item is a place where independently written Go and Python runtimes will disagree:

1. **`one r in entries`** with zero matches (undefined? absent binding?) and with more than
   one (error, first by seq, anomaly?). Alloy's `one` means "exactly one"; this uses it as a
   binder.
2. **Undefined propagation.** "Collapses as each rule declares", but there is no syntax for
   the declaration, no default, and no rule for `present(a.b)` when `a` is undefined or for
   `undefined and false`.
3. **Short-circuit versus accumulation.** When step 6 FAILs at seq 2, do family rules,
   machines and relations still run on seq 3..n, and do their anomalies print? Do
   relations run before or after the walk, and over entries that failed their MAC?
4. **`step "3a" after "2"`** in a body that is already textually ordered: two sources of
   ordering truth. Which wins, and what does `after` mean for steps inside `for e in
   entries` (per entry, or all of step 7 before any of step 8)?
5. **`not_reached "9"`** is per step, but steps repeat per entry. For N014, step 9 *did*
   run for seq 0. The expectation needs `(step, seq)` scope.
6. **`strict upgrades PASS_WITH_ANOMALY -> FAIL`**: which Step and Reason does the
   upgraded FAIL print, which anomaly wins when several exist, and does the upgrade
   short-circuit at step 12a or apply at the end?
7. **`severity anomaly`** on a relation: is it overridable by `strict`, and what marker
   string does it emit (Go maps family names to markers in `core/verdict`)?
8. **`map<string, number[0,1]>`**: neither `map` nor `number` is in the 01 §3 type list.
   DSL-009's integer-only IR constrains the *pack*, not the *input*: `classifier_scores`
   arrive as JSON floats. Go parses to `float64`, Python to `float`; a value like
   `1.0000000000000001` passes in both after rounding. Bounds need an exact decimal-string
   comparison primitive with its own conformance cases.
9. **`ts_utc` ordering**: Go `time.Parse(RFC3339)` and Python `datetime.fromisoformat`
   accept different strings (offset forms, fractional digit counts). Parsing is a primitive
   and must be specified as a grammar.
10. **`matches(x, TENANT_ID_CLASS)`** implies regular expressions. RE2 and Python `re`
    differ. Use a character-class primitive, not regex.
11. **Undeclared names in examples:** `expected_prev`, `keys`, `last_byte`, `info_for`,
    `ikm_lookup`, `session_key`, `hkdf_inputs_digest` are not in the nine primitives of
    01 §4. `ikm_lookup` is context I/O despite "no I/O".
12. **Outcomes conflate status and exit code.** `STRUCTURAL` and `CONFIG` are not R8
    statuses; the wire strings (`PASS-WITH-ANOMALY` vs `PASS_WITH_ANOMALY`) are not mapped.
    The pre-flight exit code is "not stated" in the spec (Research 01 §1), yet the example
    assigns STRUCTURAL, deciding drift by syntax.
13. **Budget.** "Steps capped proportional to input size" must count identically in every
    runtime, or one returns CONFIG while the other returns a verdict: a differential
    failure by construction. The cost model would have to be normative.

**Fix.** Under the alternative below most of these disappear, because they live in host
code where each witness answers them from the spec. Any that remain in shared data (1, 8,
9, 10, 12) get a written rule and a conformance case before any runtime consumes them.

### F6. The query planner is YAGNI. Severity: Medium

**Attacked:** 02 §6 "Complexity".

A planner that must produce identical *results* (not plans) in two languages, with a
normative cost estimate for budgeting, to replace joins that are ten lines of `map[key][]T`
in Go. Every relation in Research 01 is an equality join on a parent reference.

**Fix.** If relations are ever shared, declare the index explicitly
(`index approvals keyed_by (parent_run_id, parent_seq)`) so O(n) holds by construction and
no planner exists.

### F7. The vector language contradicts DSL-005 and DSL-002. Severity: High

**Attacked:** 02 §8, 01 §5, decisions DSL-002/005/008.

`ref("042").merkle_root` "resolves at compile time", so the open compiler must compute
Merkle roots: crypto in the compiler, against the non-goal. `rechain mac | attacker_key`
means the corruption semantics require the chain *writer*, which lives in the proprietary
TesseraSeal engine. So the open grammar carries syntax whose meaning is defined only in
closed code: an examiner can parse a vector and cannot reproduce it. That is the worst of
both licensing positions.

**Fix.** Vectors are not part of the language. Keep them as a Python library in TesseraSeal
(an embedded DSL: builders, dataclasses, one JCS, one Merkle), and publish a **negative
manifest** in JSON (case, layer, expected Status/Step/Reason/Exit/not-reached with seq
scope). The manifest is data, open, and read by the Go harness, which also retires the rule
knowledge hidden in `negative_walk.go`.

### F8. Printing the rule-pack hash in every verdict leaks the DSL into the spec. Severity: Medium

**Attacked:** 00 R3, 01 §4 outcome renderer, 01 §6.

`rulepack_sha256` in the `Verdict-Object:` line is a spec-governed output field (C5) that
only CDL-based verifiers can produce. A third-party verifier written from the spec cannot
emit it, and every whitespace edit to `rules/` churns any vector that pins full verdict
bytes.

**Fix.** Report it in a verifier-version line outside the normative Verdict-Object, or not
at all.

### F9. P0 builds a compiler front end to express constants. Severity: High

**Attacked:** 03 §4 P0.

P0's scope is constants, enums, exit codes, reason templates and step IDs: data. Yet it
builds a lexer, parser and resolver (FormBuilder's tokenizer alone took most of 26 rounds)
plus two runtimes that load tables at run time, which turns Go compile-time errors (a typo
in a constant name) into run-time lookups. And roughly half the drift register does not
move with tables at all: D3, D4, D5, D6, D11, D12, D16 and D18 are code bugs. D16 (the CLI
runs the non-conformant pipeline) and D18 (JCS sorts by UTF-8, not UTF-16) are the two most
damaging and sit in P3 and nowhere, respectively.

**Fix.** P0 is a JSON data file plus a generator that emits typed Go and Python, and the
code fixes for D16 and D18 happen now, by hand, without waiting for a language.

### F10. Phases are ordered backward on evidence. Severity: Medium

**Attacked:** 03 §4.

The migration oracle (25 `_compute.py`, the only existing byte-exact ground truth) is
consumed in P4, after four phases of rules that it would have checked. Research 01 also
warns the survey was run against a stale spec (draft.7 while Go cites §10.84 and N036+),
yet the grammar is designed now.

**Fix.** Order: re-survey against local `ffiec-public`; decide the drift register; P0 data
tables; vectors manifest and oracle next, so every later phase is tested against bytes.

### F11. "Independent" runtimes written by one model are correlated. Severity: Medium

**Attacked:** 00 R1, 03 §1 level 4.

Two runtimes written by agents of the same model family from the same IR prose share
misreadings (see F5.1, F5.3: the ambiguity gets resolved the same "natural" way). Agreement
is then weak evidence. Hand-written verifiers are subject to the same effect, but there it
is at least spread across different code rather than concentrated in one spec's gaps.

**Fix.** Wherever a behavior is shared, pin it with a conformance case, not a paragraph.

### F12. Maintenance budget is not stated and does not fit one maintainer. Severity: High

**Attacked:** 03 §5 risk 6, the six phases.

Owned artifacts under v0: language spec, IR schema, IR semantics spec, compiler (seven
packages plus formatter, linter, generators), fuzz harness, goldens, Go runtime, Python
runtime, .NET runtime, runtime conformance corpus, primitive bindings per host, rules,
vectors. Every language change touches five of them. None of this is required by any
examiner, customer or spec editor.

**Fix.** Adopt the alternative. Set a kill criterion: if a textual language is ever
proposed again, it must replace more lines than it adds, counted.

### What v0 gets right and the alternative keeps

The drift register and R10 (one answer per question). `@spec` citations with quote hashes
and a build-time verbatim check. Pre-split reason templates with explicit `:quoted`. Three
corruption layers and author-declared expectations. Closed-by-default schemas. The
fail-closed, fuzz-contract, golden-byte discipline. The mutation check (it works on data
too: delete a table row, flip an enum member).

## Minimum viable alternative

**Shape: typed data, generated constants, small table engines, hand-written procedures.**

1. **`rules/ffiec.rules.json`**, one closed JSON document (JSON because it is the only
   format both Go and Python parse with the standard library). Holds: constants; enums with
   wire strings; outcome statuses with exit codes; the §7 step table (id, order, exit,
   pre-split reason template with slot formats); family schemas (field, type from a closed
   set, required, enum ref, open prefix); conditional-presence rows
   (`when field in {...} require [fields]`), a fixed shape, not an expression; state
   machines (states, start, transitions, terminal, ordered checks with reasons, key
   fields). Every entry carries `spec` and `quote_sha256`.
2. **`rulesgen`** (Go, stdlib only): loads with `DisallowUnknownFields`, resolves refs,
   checks template slots and spec quotes, and emits checked-in `rules_gen.go`,
   `rules_gen.py` (later `Rules.g.cs`) plus Markdown tables and the drift report. CI
   regenerates and diffs; a tripwire test forbids hand-held copies.
3. **Table engines, hand-written per language:** a state-machine walker and a
   family-schema checker that interpret the fixed-shape tables. .NET already has one
   (`StateMachine.cs`).
4. **Procedures hand-written per language:** the §7 walk, relations (§10.84, adjuster
   bidirectional refs), backfill, bordereau guards, the routing map-key check. Reasons,
   exit codes and step IDs come from the generated data, so the strings cannot drift, while
   the logic stays independent, which is what C2 requires.
5. **Vectors:** a Python library in TesseraSeal plus the open JSON negative manifest (F7).
6. **Escape hatch, gated on counted evidence after P2:** if more than roughly 40 genuine
   cross-field expressions remain that tables cannot hold, add a closed JSON expression AST
   (PredicateSpec-style: `eq`, `in`, `present`, `and/or/not`, no binders) evaluated by
   about 300 lines per language. Still no grammar.

**Phasing:** P0 re-survey, drift decisions, rules JSON for constants, enums, exits, reasons,
steps; fix D16 and D18 by hand. P1 schemas and machines plus table engines. P2 negative
manifest and oracle migration of `_compute.py`. P3 hand-written relations and §7 extras in
Go and Python. P4 .NET consumes the JSON.

### Cost comparison (rough, source plus tests)

| Item | v0 (CDL) | Alternative |
|---|---|---|
| Grammar, lexer, parser, resolver, type checker, lowering, formatter, linter, planner | 8–12k Go + 12–20k tests (FormBuilder ratio) | 0 |
| Data validator and generator | inside compiler | 0.8–1.2k Go + ~1k tests |
| Normative prose | language spec + IR semantics spec, both large | JSON shape doc, short |
| Runtimes | 3 × 3–5k, plus a runtime conformance corpus | table engines 3 × 0.4–0.6k |
| Procedural rules (§7, relations, guards) | written once in CDL, plus primitive bindings per host | 3 × 1.5–3k, hand, independent |
| Vectors | vector language in compiler + proprietary engine | Python library + JSON manifest |
| **Total owned code** | **~35–55k lines** | **~8–14k lines** |
| Time to first drift closed | end of P0 after a front end exists | first week |
| Witness independence | one transcription, N interpreters | shared tables, independent logic |

The break-even for a textual language is when the procedural code written three times
exceeds a compiler, three runtimes and two normative specs. At ffiec's current size (about
860 lines of Go rule logic, about 2.5k rule-shaped lines in .NET), that point is roughly an
order of magnitude away.
