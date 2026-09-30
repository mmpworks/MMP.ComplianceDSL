# 00: Charter (v1)

Status: **v1, after red team.** Date: 2026-09-30. Supersedes the v0 design (unpublished; summarized in `decisions.md`). The
dispositions are in `../redteam/dispositions.md`.

## What changed from v0, in one paragraph

v0 proposed a custom textual language (CDL) with a compiler, an IR and interpreters in each
language. The interpreters would have *run* the verifier's rules, including the §7 walk.
All three red teams rejected that on the examiner path, each from its own lens:

- **Assurance (RT-1).** One transcription run by N interpreters is one witness. The
  crypto composition would end up living in data, and a valid pack could quietly weaken
  verification.
- **Engineering (RT-2).** CDL is seven languages. It would take about 35–55k lines owned
  by one maintainer. The design's own examples already misread the Go code.
- **Governance (RT-3).** The pack would become a second channel for changing the spec,
  owned by a vendor.

v1 keeps what v0 got right: one source for every string and table, drift decisions made
once, citations to the spec, and discipline around golden bytes. It drops the language.

**v1 design in one line:** typed data, generated constants, small table engines, and
procedures written by hand.

## Problem (unchanged)

The chain-of-custody rules exist as hand copies in four places: the spec prose, Go, .NET,
and 25 Python `_compute.py` scripts. A fifth consumer, the TesseraSeal VectorCompiler, is
planned. The copies have drifted in 18 places (`drift-register.md`). The spec requires more
than any one language implements.

## Goal

1. **One closed, typed data source for everything that is data in the spec.** That means:
   - constants;
   - enumerations and their wire strings;
   - outcome statuses and exit codes;
   - §7 step IDs and their order;
   - reason templates;
   - attribute-family field schemas;
   - rows that state conditional presence;
   - state-machine tables.
2. **One generator that emits checked-in typed code** for Go and Python, and later C#.
   Every string and table in every implementation comes from it. Hand-held copies are
   forbidden by a tripwire test.
3. **Small table engines written by hand in each language:** a family-schema checker and a
   state-machine walker.
4. **Procedures stay code, written by hand and independently in each language.** These are
   the §7 walk, the crypto composition, and relations between events. They read their
   strings from the generated data and decide their logic themselves.
5. **An open conformance toolchain for vectors:**
   - a JSON negative manifest;
   - an open reference materializer.

   Commercial tools, including MMPWorks' TesseraSeal, may build on these. The open corpus never depends on them.

## Non-goals

- **A textual language, a grammar or a query planner.** There is one escape hatch, gated on
  counted evidence (DSL-016).
- **Rules that execute on the examiner path.** Data never decides control flow in the §7
  walk.
- **Cryptography, or crypto composition, in data.** No data says which bytes get MACed or
  compared.
- **Settling spec-meaning questions in this repo** (see R10).
- **Replacing the spec.** This repo holds an *unofficial transcription*. The prose governs.

## Requirements (v1)

**R1. Defect independence.** Shared artifacts are strings and tables only. Every decision
is logic written by hand in each language. Where practical, the Python table engines and
procedures are written from the spec prose without reading the Go code. Different agents or
models write them (RT-2 F11).

**R2. The examiner path is open and uses only the standard library.**

- The generator is Go standard library only.
- The generated Go code has no imports beyond the standard library.
- The whole path builds offline.

**R3. Reproducible.** `rules.json` → generator → the generated files is a fixed point, and
CI regenerates and diffs. The generated files embed the SHA-256 of `rules.json` as a
constant. The verifier prints it on a **diagnostic line after** `Verdict-Object:`, never
inside it, because the verdict schema is closed (RT-3 F3).

**R4. Traceable.** Every entry carries `spec` (section) and `quote_sha256`.

- The quote check renders each reason template with its **real** formatting (quoting
  included) against the spec's own examples.
- It is a lint. It is not an assurance claim (RT-1 F5).

**R5. Fails closed. There is one rule for undefined values.**

- *Cannot evaluate* takes the severity of the rule that could not be evaluated: FAIL for
  FAIL rules, an explicit "cannot evaluate" anomaly for anomaly rules.
- There is no per-rule policy and no vacuous truth for `exists` (RT-1 F6).

**R6. A pinned input parsing profile.** Inputs are parsed as I-JSON (RFC 7493), with a
conformance case for each of these rules (RT-1 F7):

- reject duplicate keys;
- reject integers above 2^53;
- match keys case-exactly;
- accept only strict UTF-8.

**R7. Safety invariants live in host code.** Each host hard-codes checks that data cannot
override (RT-1 F3):

- FAIL maps to a non-zero exit code;
- step 7 < 8 < 9;
- the witness-mode skip set is a subset of {7, 8, 9};
- strict mode only ever upgrades severity.

The build fails if the generated data would violate any of them.

**R8. Outcome fidelity.** The status set is `PASS`, `FAIL`, `PASS-WITH-ANOMALY`,
`PASS-STRUCTURALLY` and **`PASS-CUSTOMER-DISCLOSURE`**. Each step has a string ID **and an
explicit integer ordinal**; string IDs are never sorted (RT-1 F8).

**R9. Migration oracle, early.** The 25 `_compute.py` outputs are reproduced byte for byte
**in P1**, not at the end (RT-2 F10).

**R10. Disputed entries have no answer.** Drift items are classified into two kinds:

- `impl-bug`: decided here, and fixed by hand in code.
- `spec-meaning`: goes to a public `spec-proposal`. The entry is marked `disputed`, and
  the generator **refuses to emit it**. A phase that needs a disputed entry cannot ship
  until the spec settles it (RT-3 F1).

**R11. Honest about independence.** The implementations MMPWorks writes are *reference
implementations*. They do not count toward the governance requirement of "three independent
implementations" (RT-1 F12, RT-3 F4). The foundation gate needs outside implementers, and
the open toolchain (Goal 5) exists so that outside implementers are possible.

## Success measures

- Zero hand-held copies of constants, enums, reason strings, exit codes, step IDs or
  transition tables in Go or Python. A tripwire test enforces this.
- Every `impl-bug` drift item is closed. Every `spec-meaning` item is filed as a spec
  proposal.
- 25 of 25 `_compute.py` outputs are reproduced byte for byte, or each difference is logged
  and decided.
- A spec coverage matrix lists every MUST and every normative reason string, each mapped to
  a table row, a test or "not yet transcribed".
- Total owned code is about 8–14k lines (RT-2's estimate), against about 35–55k for v0.
