# 03: Phases and verification (v1)

Status: **v1, after red team.** Supersedes the v0 design (unpublished; summarized in `decisions.md`).

## 1. Phases

Every phase ships something usable on its own and has an exit gate.

**P0: ground truth and hand fixes.** Mostly outside this repo; about 1–2 weeks.

1. **The maintainer pushes the current local spec corpus**, so everything is designed against
   the real spec and corpus. v0 was designed against a stale May copy (the draft.7 skew;
   RT-2 F10).
2. **Re-run the rule inventory** against that corpus and refresh `drift-register.md`. Classify every drift item as `impl-bug` or
   `spec-meaning`. File each `spec-meaning` item as a public spec proposal.
3. **Hand-fix the defects that no tables can fix,** in `ffiec`:
   - **D16.** Wire the CLI to the spec walk (`auditwalk.go`) instead of the legacy
     pipeline.
   - **D18.** JCS key sort by UTF-16 code units.
   - **Constant-time comparison** at `auditwalk.go:359` and `:383`.
   - **Step 11 must not silently skip.** At `auditwalk.go:195-201` it currently returns
     PASS without running.
4. **Build the spec coverage matrix.** Extract every MUST sentence and every normative
   reason string from the spec. Each item gets one mapping: an implementation, a test, or
   "not transcribed". This is what a mutation check cannot show (RT-1 F4).

*Exit gate:*
- the public spec corpus matches the maintainer's working copy;
- the drift items are classified;
- D16 and D18 and the constant-time fixes are merged;
- the coverage matrix is v1.

**P1: tables for strings and constants, and the byte oracle.** About 2–3 weeks.

1. `spec/rules.schema.json`, `rulesgen`, and the generated Go and Python code for:
   - constants
   - enums
   - statuses
   - steps and their reason templates
2. **ffiec deletes its hand-held copies**, and the tripwire test lands.
3. **The open Python primitive module** replaces the duplicated helpers in the 25
   `_compute.py` scripts. Run the byte oracle right away and log every difference as a
   drift item. Known in advance:
   - 003's non-JCS canonicalization;
   - 008's Python exception names;
   - the CRLF writes.

*Exit gate:*
- zero duplicated constants and strings in Go and Python;
- the oracle diff is closed or logged;
- the R7 invariants are enforced in `rulesgen` and in each host.

**P2: families and machines.** About 3–4 weeks.

1. Family tables and conditional rows, plus the family checker in Go and in Python.
2. Machine tables plus the walker in Go and in Python. ffiec gains §10.43, §10.46, §10.50
   and §10.55 verification. This is new capability, not a port.
3. Engine conformance cases, and a Go/Python differential run in CI.
4. **The DSL-016 count:** how many cross-field conditions remain hand-written.

*Exit gate:*
- vectors 037, 038, 040 and 048 and their negatives are live in Go;
- the differential is green;
- the drift items D11–D13 have been decided or filed as spec proposals.

**P3: the negative manifest and the open materializer.** About 2–3 weeks.

1. `spec/negative-manifest.schema.json`: three layers, a triple outcome, and
   `not_reached` scoped by `(step, seq)`.
2. The open reference materializer. It reproduces every negative the public corpus
   describes.
3. **TesseraSeal Task #24 is rebuilt on top of it.**
4. The negative index is generated.

*Exit gate:*
- every negative can be materialized by open tools;
- the Go gate runs them live;
- `_compute.py` is retired.

**P4: procedures, hand-written in Go and Python.** Ongoing. This covers the §7 extras no
language implements yet:

- 3a and 12a;
- dual-algorithm cases (a) through (e);
- the `key_versions` cross-check;
- step 12;
- witness mode and customer-disclosure mode;
- late-binding;
- unknown-kind handling;
- the `Verdict-Object` line.

It also covers relations: §10.84, §10.45 and §10.49.

**P5: .NET.** `Rules.g.cs` goes into Herald.Compliance. The DSL-016 decision is made here.

Durations assume one maintainer working through agents. They are rough, and the P0
re-survey will revise them.

## 2. How the library proves itself

1. **Schema and invariant gates in `rulesgen`:**
   - closed keys, no duplicate keys, all references resolved;
   - template slots bound;
   - R7 invariants hold;
   - no `disputed` entry is emitted.
2. **A generation fixed point.** CI regenerates and diffs. The generated code has
   byte-golden snapshots, not substring checks.
3. **Tripwire tests in each host:**
   - no hand-held reason prefixes, enum wire strings or transition tables outside the
     generated file;
   - no control flow in the generated file.
4. **Engine conformance plus differential runs.** The Go and Python engines must produce
   identical findings on every case and every spec vector.
5. **Input-profile conformance.** Every rule in the I-JSON profile has a case, each
   rejected identically in Go and Python.
6. **Fuzzing.** Go native fuzzing of `rulesgen` and the engines, under a fuzz contract:
   the only allowed outcomes are a value or a declared error.
7. **Mutation check over data.** Delete a row, flip an enum member or swap two transitions,
   and some case must fail. It is paired with the coverage matrix, because mutations only
   test what was transcribed.
8. **The quote lint.** Render each template with sample values, including its real quoting,
   and look for it in the pinned spec. It is a lint, not a guarantee.

## 3. Budget

This is RT-2's estimate, adopted:

| Item | Lines |
|---|---|
| `rulesgen` | about 1k, plus tests |
| Table engines | about 0.5k per language |
| Materializer and primitives | about 1.5k |
| Procedures written by hand | about 1.5–3k per language, mostly already in ffiec |

**Total owned: about 8–14k lines.** Against v0's 35–55k, that is where most of the savings
come from.
