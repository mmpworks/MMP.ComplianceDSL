# Red-team dispositions

Three reviews were run in parallel on design v0, each through a different lens:

- **RT-1:** assurance and independence
- **RT-2:** language and engineering
- **RT-3:** governance, licensing and adoption. The RT-3 report itself is kept private because it covers legal and IP questions; its findings and dispositions are listed below.

Each finding has one of four dispositions:

| Disposition | Meaning |
|---|---|
| **Accepted** | v1 changes because of the finding. |
| **Partial** | v1 changes, but not all the way the finding asked. The reason is given. |
| **Rejected** | The finding is not acted on. The reason is given. |
| **Maintainer** | The finding needed the maintainer's decision. |

**Overall.** All three reviews independently reached the same conclusion: a shared,
interpreted rule pack must not run the examiner's verification. v1 (DSL-011) adopts RT-2's
minimum-viable alternative, with RT-1's safety requirements and RT-3's governance
requirements on top.

## RT-1: assurance and independence

| # | Sev | Finding | Disposition |
|---|---|---|---|
| 1 | Crit | One rule pack is one witness wearing three coats | **Accepted.** DSL-011: logic is written by hand in each language, and data holds only strings and tables. R1 adds writing the Python code from the spec prose with a different agent or model. |
| 2 | Crit | Crypto wiring in data; no constant-time comparison; the MAC input is wrong in the v0 example | **Accepted.** DSL-014. Constant-time fixes at `auditwalk.go:359,383` are in P0. |
| 3 | Crit | A valid pack can weaken verification; PASS-CUSTOMER-DISCLOSURE missing; step 11 silently skipped | **Accepted.** R7 and DSL-019 add host invariants. The status is added (R8). The step 11 fix is in P0. |
| 4 | High | The mutation check can't see rules that were never transcribed | **Accepted.** The spec coverage matrix is built in P0. |
| 5 | High | The quote check is gameable and blind to quoting | **Partial.** The lint now renders real quoting, and it is explicitly downgraded to a lint (R4). A self-pinned spec remains a limit until the spec is pinned by a public tag. |
| 6 | High | Three-valued logic defaults to fail-open | **Accepted.** DSL-017 sets one rule: no per-rule policy and no vacuous truth. The general expression language is gone. |
| 7 | High | Input parsing is unspecified (duplicate keys, 2^53, case folding); Go has no schema validator | **Accepted.** DSL-018 sets the I-JSON profile with conformance cases. `rulesgen` enforces the schema itself, with no validator dependency. |
| 8 | High | Interpreting rules is harder for an examiner to audit; string step IDs sort wrongly | **Accepted.** The walk stays a short hand-written list, and steps get integer ordinals. |
| 9 | Med | Self-attested hash from go:embed; can the pack be swapped? | **Accepted.** DSL-015: the data is compiled in, with no `--rules` flag. The hash is informational and printed after the verdict. |
| 10 | Med | An input-scaled budget lets the examinee turn FAIL into a config error | **Accepted.** Budgets are fixed caps and produce a distinct STRUCTURAL reason (01 §6). |
| 11 | Med | Weakly typed primitive binding | **Moot.** No primitives are bound through data. |
| 12 | Med | "Independent" means one vendor | **Accepted.** DSL-022 and R11: these are reference implementations, and outside implementers are the goal. |
| 13 | Low | Closed-by-default families contradict forward compatibility | **Accepted.** `unknown_fields` follows the policy of each spec section. |

## RT-2: language and engineering

| # | Sev | Finding | Disposition |
|---|---|---|---|
| F1 | Crit | A shared pack defeats independence | **Accepted.** Same as RT-1 #1. |
| F2 | Crit | The §10.84 example misreads Go in six ways; the 037 quoting is dropped | **Accepted.** §10.84 stays hand-written (P4). Examples are shapes only until the P0 re-survey. The byte oracle moves to P1 so misreadings surface early. |
| F3 | High | The DSL-003 dismissals are partly wrong; Starlark missed; wrong baseline | **Accepted.** Recorded in the DSL-003 re-evaluation. |
| F4 | High | CDL is seven languages | **Accepted.** The language is dropped. |
| F5 | High | 13 underspecified semantics | **Accepted, mostly moot.** Six are removed along with the language. The rest are specified in 02 §3 (engine semantics), R5, R6 and the manifest's `(step, seq)` scope. |
| F6 | Med | The query planner is YAGNI | **Accepted.** Removed. |
| F7 | High | The vector language needs crypto in the compiler; open grammar, closed semantics | **Accepted.** An open manifest plus an open materializer (DSL-021). |
| F8 | Med | The rule-pack hash in the verdict leaks into the spec | **Accepted.** Printed on a diagnostic line (DSL-015). |
| F9 | High | P0 builds a front end to express constants; D16 and D18 unaddressed | **Accepted.** No front end at all. D16 and D18 are fixed by hand in P0. |
| F10 | Med | Phases are ordered backwards on evidence; the survey was run on a stale spec | **Accepted.** The re-survey and the oracle move first. |
| F11 | Med | Runtimes written by one model are correlated | **Partial.** A different agent or model writes from the prose. The correlation is reduced, not eliminated, and is disclosed (R11). |
| F12 | High | Maintenance does not fit one maintainer | **Accepted.** About 8–14k lines; the budget is in 03 §3. |

## RT-3: governance, licensing and adoption

| # | Sev | Finding | Disposition |
|---|---|---|---|
| F1 | Crit | The pack becomes a second channel for changing the spec | **Accepted.** DSL-020 and R10: `spec-meaning` drift goes to public proposals, and disputed entries are not emitted. |
| F2 | Crit | Generated tables would invert authority | **Accepted.** Generated docs are for *checking* the prose and never go into `spec/`. The document declares itself unofficial (DSL-023). |
| F3 | High | The hash in the verdict changes the closed schema | **Accepted.** DSL-015. |
| F4 | High | The foundation gate would be met by one author | **Accepted.** DSL-022. |
| F5 | High | The disclosure statement was inaccurate; product names and vendor contacts appeared in spec text | **Fixed 2026-09-30.** Conflict-of-interest disclosure added; product names and vendor contact addresses removed (branch `claude/busy-faraday-9sf328` in the spec and verifier repos). |
| F6 | High | A proprietary corpus generator conflicts with the attestation registry | **Accepted.** DSL-021 adds the open materializer. |
| F7 | High | Keeping the rules syntax private conflicts with an open verifier | **Resolved 2026-09-30.** The rules format is public so the community can improve it (DSL-012). This repo is published from a clean history. |
| F8 | Med | "Private for now" would end at P0 anyway | **Resolved.** The repo is published now. |
| F9 | Med | Patent implications of Apache-2.0 | **Maintainer.** Handled privately. |
| F10 | Med | Ownership and provenance of agent-written code; contribution terms | **Maintainer.** Contributions follow the spec repo's model: Apache-2.0 inbound = outbound, no CLA. A provenance log is kept. |
| F11 | Med | "FFIEC rule pack" implies official status | **Accepted.** DSL-023. |
| F12 | Med | The spec is pinned to a vendor copy; the draft cadence breaks governance | **Accepted in part.** The spec is pinned by repo tag and commit. The cadence is the maintainer's. |
| F13 | Low | Competitors won't adopt CDL | **Accepted, moot.** There is no language to adopt, and the corpus stays complete without the tooling. |
