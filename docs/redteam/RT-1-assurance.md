# RT-1: Assurance, cryptographic soundness, and witness independence

> Note on publication: this review cites "Research 01/02", internal research notes that are not published. The drift items it cites are in `docs/design/drift-register.md`. The v0 design it reviews is summarized in `docs/design/decisions.md`.

Reviewer lens: a skeptical FFIEC examiner and a cryptographic auditor. Target: design v0
(`docs/design/00`–`03`, `decisions.md`). Date: 2026-09-30.

## Verdict

**Do not ship v0 as designed on the examiner path.** The design names the common-mode risk
(03 §5 risk 1) and then removes the only mechanism that has ever caught one: independent hand
transcriptions that disagree. The 18-item drift register is not evidence that duplication
failed. It is evidence that independent witnesses worked. v0 collapses rule semantics to a
single transcription, written by one author through agents. That author also writes the
expected outcomes and both runtimes. The result counts as three witnesses on paper and one
in fact. v0 also puts the **composition** of the cryptography into interpreted data: which
bytes are MACed, which previous hash feeds the MAC, and which comparisons are constant-time.
DSL-005 claims the opposite. And nothing in the verifier binary stops a pack from dropping,
reordering or downgrading a step. The design's own worked example in 02 §6 already
mistranscribes the one rule that has a Go reference (Finding 1). P0 (shared constants and
tables) can ship once Findings 3, 5 and 7 are fixed. P3 (the §7 walk as pack data) should
not ship in any form that deletes the hand-written walk.

## Findings

### 1. The rule pack is one witness wearing three coats. **Critical.** Attacks DSL-001, DSL-007, 00 R1, 01 §1.

**Claim.** "Share data, not code … one bug would pass everywhere" (01 §1). The claim is that
separate engines over a shared pack stay independent witnesses.

**Failure.** The engines are independent only for *interpreter* bugs. For *rule* bugs,
meaning a mistranscription of the spec, every runtime executes the same wrong rule. The
author's declared expectation (DSL-007) comes from the same reading of the spec. The Go and
Python runtimes agree. The differential stays green. The hand-written Go code, the one
artifact that would have disagreed, has been deleted (01 §4, "hand tables are deleted, and a
tripwire test forbids new ones").

**Evidence: the design's own example.** 02 §6 transcribes §10.84 and diverges from the Go
reference (`verifier/internal/verify/communication_preapproval.go`) in four ways:

- **When the check fires.** CDL runs `require exists(approval)` for every retail
  communication. Go returns no anomaly when no send exists (`checkOneCommunication`, the
  `!hasSend` branch: "a communication approved but not yet sent is a normal pending state").
  The CDL rule therefore raises false anomalies on every pending communication.
- **Which seq is reported.** CDL renders `{c.seq}`, the communication's seq. Go renders
  `sendSeq`. The reason bytes differ.
- **What happens when the send is absent.** `approval.signed_at_utc <= send.applied_at_utc`
  evaluates to `undefined`, and the 02 §6 policy "violation" makes that a *second* false
  anomaly.
- **What `one` means.** `one r in entries` is undefined: exactly one, or first match? Go
  takes the first match in file order (`findPrincipalApproval`), so a later duplicate
  approval changes the answer in only one of the two readings.

**The compiler is shared too.** `cdlc` is shared executable code whose output every witness
consumes. A bug in the lower stage ("fixes a canonical order for declarations, and numbers
the steps", 01 §3.6) reaches every runtime at once. R1 ("no executable code is shared") is
violated by construction.

**Fix.** Reverse DSL-001's direction on the examiner path.

- **Make the pack an oracle, not the executor.** Keep a hand-written Go §7 walk and hand-written
  family checks. Run pack-driven Python and hand-written Go differentially over the corpus.
  Disagreement then means that either the transcription or the code is wrong, which is what
  the drift register already demonstrates happening.
- **Tables are safe to share.** Share constants, enum wire strings and reason templates
  (P0). Do not share predicates, procedures or crypto composition across all witnesses.
- **Would I ship DSL-001 as written? No.** I would ship "share tables and vectors; keep at
  least one hand-transcribed executor of rules per spec version."

### 2. DSL-005 is a category error: the crypto composition lives in the pack. **Critical.** Attacks DSL-005, 00 Non-goals, 02 §7.

**Claim.** "Cryptography inside the language" is a non-goal, and CDL "only names" primitives.

**Failure.** The primitives are native, but *how they are wired* is pack data. Each of the
following is a one-token data edit that no primitive audit sees:

- **Which bytes are MACed.** 02 §7 step 9 uses `hmac_sha256(session_key(ikm, …), expected_prev ++ jcs(e.event))`.
  That re-canonicalizes a parsed object. The spec (§4.1 item 7) requires canonical bytes
  that *exclude the chain-stamp fields*. Go instead MACs the pinned `event_canonical_hex`
  and says so deliberately: "never over a re-canonicalization" (`auditwalk.go:66-68`,
  `:376`). Three answers, and the pack picks one silently. Re-canonicalizing in the verifier
  also re-exposes the D18 UTF-8 key-sort bug (`core/jcs/jcs.go:91-104`) that Go's walk
  currently sidesteps.
- **The `expected_prev` footgun.** The spec (§7 step 9) says the MAC input "MUST use
  `expected_prev_hash` … NOT `entry.prev_hash`". In CDL this is a path choice:
  `expected_prev` versus `e.prev_hash`. While step 6 holds, the two are equivalent mutants,
  so no corpus case distinguishes them (see Finding 4).
- **Constant-time comparison.** §4.1 item 4 says the verifier "MUST use a constant-time
  equality primitive for both the fingerprint check and the MAC check". The IR's generic
  `{"k":"eq"}` (02 §9) cannot know that its operands are secret-derived. Go already fails
  this requirement: `auditwalk.go:359` and `:383` compare hex strings with `!=`.
- **Binding checks read unauthenticated data.** Step 4 (`auditwalk.go:312-318`) compares
  the tenant from the *decoded* `Event` map. The MAC covers `EventCanonicalHex`, and no code
  checks that the two correspond. A pack inherits this pattern freely.
- **Downgrade by dispatch.** `sign_payload(version, fields)` dispatches on the seal's own,
  attacker-controlled `sign_payload_version`. The set of versions the verifier accepts is a
  security parameter, and it lives in data.

**Fix.**

- **Make steps 2 and 6–11 sealed native steps.** They should be host functions with fixed
  signatures, for example `verify_entry_mac(header, entry, ikm, expected_prev) -> ok|fail`.
  The pack may reference them by ID and render their reasons. It may not wire their inputs.
- **Add a `ct_eq` primitive and secret types.** Give `Mac`, `Fingerprint` and `Key` their
  own nominal types, and have the type checker reject `eq` on them.
- **Pin the accepted `sign_payload` versions in the host,** not the pack.

### 3. A pack can weaken verification, and the verifier cannot tell. **Critical.** Attacks 00 R5, 01 §4 Loader, 02 §7.

**Failure.** The loader checks the schema version, the hash and unknown node kinds. All of
the following are *well-formed* packs that pass those checks:

- **Exit codes.** Change the `outcomes` table's `FAIL exit 1` to `exit 0` (02 §7). Every
  scripted examiner pipeline goes green.
- **Witness skips.** Widen the witness mode to `skips "7","8","9","10","11"`. The spec
  says: "An implementation that skips additional steps in witness mode (e.g., omitting Merkle
  recomputation) is non-conformant" (spec §7).
- **Strict mode.** Remove `strict_fail` from the co-signed case (e). Strict mode stops
  failing a Severe finding (spec §7 11(e)).
- **Severity.** Re-declare a step-level rule as `severity anomaly`. The result becomes
  PASS-WITH-ANOMALY with exit 0.
- **Step order.** Drop step "8" or change the `after` edges. The spec (§7, "Step ordering")
  makes 7 → 8 → 9 normative precisely so that botched rotations are not misreported as MAC
  failures.

**Content addressing does not help.** A content address proves *which* pack was used, not
that the pack is correct. The same trust domain produces both the pack and its hash.

**Evidence that this already happens.** `WalkAuditFileWithKey` silently skips step 11 when
`pub == nil` and returns plain `PASS` (`auditwalk.go:195-201`). The spec requires step 11 in
witness mode and the status `PASS-STRUCTURALLY`. A pack is exactly the place where a skip
like this becomes invisible.

**The outcome model is also incomplete.** R8 lists four statuses. The spec has a fifth,
`PASS-CUSTOMER-DISCLOSURE, institution-IKM verification skipped` (spec §7, customer-disclosure
mode), and says implementations "MUST NOT silently fall back" between modes.

**Fix.** Add a **hand-written pack guard** in each host, roughly 150 lines of native code
that are not generated. It checks:

- the required step IDs per mode, taken from spec §7's witness and customer-disclosure
  paragraphs;
- the exact skip set per mode;
- the 7 < 8 < 9 order;
- a closed status set, with PASS-CUSTOMER-DISCLOSURE added;
- that the exit-code map equals §10.12;
- that strict-fail marks cover every spec "Under `--strict`: FAIL" case.

A pack that violates any of these is refused with exit 3. This guard is the independent
witness for the pack itself.

### 4. The mutation check measures corpus coverage of the pack, not pack coverage of the spec. **High.** Attacks 03 §1.6.

**Failure.**

- **Omissions are invisible.** "Delete the rule or invert its predicate" only works on rules
  that exist. A §7 MUST that was never transcribed has nothing to mutate. Research 01 says
  the rules the spec requires but nothing implements *outnumber* the implemented ones.
- **Too few operators.** Delete and invert miss: reordering; path substitution
  (`expected_prev` to `e.prev_hash`); argument swaps; flipping the undefined-policy;
  widening a mode's skip set; unmarking `strict_fail`; downgrading severity; editing
  constants; and `==` to `in`.
- **Equivalent mutants are hidden.** The footgun edit survives every corpus case while
  step 6 holds, and the harness would report it as "not covered". Nobody reviews that
  report.

**Fix.**

- **Build a spec-side coverage matrix.** Extract every §7 step, every MUST sentence in §4
  and §7, and every backticked reason string from the pinned spec. Each must map to a rule,
  a sealed native step, or a signed waiver. Make the build fail on an unmapped row.
- **Extend the mutation operators** to the list above.
- **Record surviving mutants** and require a named human disposition for each.

### 5. The spec-quote check can be gamed and does not check semantics. **High.** Attacks 00 R4, 02 §1–2, 03 §1.5.

**Failures.**

- **(a) The anchor is self-issued.** The pinned text `spec/pinned/draft-7.md` and its
  `sha256 "…"` sit in the header of the same repo, edited by the same author. The canonical
  spec "is not on GitHub" (03 §5 risk 5). Research 01 already records version skew between
  Go and the local spec copy.
- **(b) Wildcard slots absorb the drift the design claims to settle.** `:quoted` renders the
  `"` characters *inside* the slot, and the check treats slots as wildcards. So the check
  cannot tell `format_version v1.1 …` (N022 `expected.json`) from `format_version "V1" …`
  (N023). That is D2, which 02 §7 claims templates "settle".
- **(c) It checks reason strings, not predicates.**
  `step "9" : require true else FAIL "payload_hash MAC mismatch at seq {e.seq}"` passes.
- **(d) The match is too loose.** "Each literal part must appear" is satisfied by fragments
  from different sentences. Nothing binds a match to the `@spec` section it cites.
  `quote_sha256` in the IR is opaque to an examiner reading the pack.
- **(e) Review by the same process misses drift.** Research 01 attributes the D7 suffix
  violation to .NET only. Go's own step-4 reason also breaks the `: ` rule:
  `"cross-chain lift detected at seq %d (event.tenant_id mismatch)"` (`auditwalk.go:315`).
  The inventory also omits a spec self-contradiction. In witness mode with no IKM, spec
  §7's witness-mode paragraph says `PASS-STRUCTURALLY` and "Fail-closed semantics" says
  `PASS-WITH-ANOMALY`. R10 will pick one of those silently.

**Fix.**

- Pin the spec by a signed tag or commit of `ffiec-public`, verified by `cdlc`. Do not pin
  by a hash in the source header.
- Match each template contiguously against one backticked reason span inside the cited
  section. Treat quote characters as literal, never as part of a slot.
- Store the quote text and its byte offset in the IR, not only its hash.
- Rename the check to "traceability". It must never be described as a correctness check.
- Add the Go step-4 suffix and the witness-mode contradiction to the drift register.

### 6. Three-valued logic defaults to fail-open. **High.** Attacks 00 R5, 01 §4, 02 §4 and §6.

**Failures.**

- **Rules choose their own policy.** Each rule declares what `undefined` means (02 §6). The
  spec leaves no such choice: "If any step cannot be evaluated unambiguously, the affected
  unit is reported as FAILED" (spec §7, "Fail-closed semantics").
- **Guards can skip requirements.** In the §10.55 example (02 §4), `when outcome == modified
  … else require absent(...)`: if `outcome` is `undefined` (missing, or an unknown enum), is
  the `when` false, true or skipped? The conventional reading skips, which means pass.
- **Kleene `or` passes.** `undefined or true` is `true`, so a disjunct in a `require` passes
  over an unevaluable crypto check.
- **Empty quantifiers are vacuously true.** `all e in entries` over an empty list is true.
  The spec treats the empty file as a structural FAIL (exit 2).
- **The fail-opens being migrated are real.** `decodePreapprovalViews` silently `continue`s
  past undecodable entries (`communication_preapproval.go`, the two `continue`s). Its step-4
  counterpart fails closed only by accident, because `eventTenantID` returns `""`. A
  faithful transcription carries both over.

**Fix.**

- In step context, `undefined` means FAIL, and no rule can override it.
- `when` over `undefined` counts as a violation.
- The compiler rejects `or`, `not` and quantifiers over `bool?` inside `require`, unless an
  explicit, reviewed policy is attached.
- Empty-domain quantifiers need an explicit `vacuous ok|fail`.
- Per-rule policies are allowed only for `severity anomaly`, and the pack guard (Finding 3)
  enforces that.

### 7. The input data model and pack parsing are unspecified, so parser differentials follow. **High.** Attacks 01 §2 `ir.schema.json`, 01 §4 Loader, 02 §9.

**Failures.**

- **Input parsing is undefined.** The IR semantics say nothing about how fixture bytes
  become values that paths address:
  - duplicate keys;
  - integers above 2^53 in *input* (the IR caps only its own literals; Go's `interface{}`
    decode produces `float64`, while Python produces a bigint);
  - invalid UTF-8;
  - the difference between missing and null.
- **The pack loader cannot enforce the schema it claims.** Go's standard library has no
  JSON Schema validator, so "`additionalProperties: false` everywhere" is aspirational in
  the stdlib-only runtime.
- **Duplicate keys split review from execution.** `encoding/json` takes the last value for a
  duplicate key and matches struct fields case-insensitively, so `"Require"` binds to
  `require`. A pack containing duplicate `require` keys hashes fine and shows one value in a
  reviewer's diff, while the runtime executes the other.

**Fix.**

- Make an I-JSON input profile normative, and have every runtime reject duplicate keys,
  lone surrogates, and integers above 2^53.
- The loader decodes through a token stream that rejects duplicate and case-variant keys.
- The loader requires `bytes == JCS(parse(bytes))` before it hashes.
- Write the closed decoder by hand and test it against a hostile-pack corpus.

### 8. Interpreting rules makes the verifier harder to audit. **High.** Attacks 00 R2, 01 §4 and §3.

**Today.** An examiner reads `auditwalk.go:176-202`. It is a 26-line step ledger that
follows §7 top to bottom.

**After v0.** The examiner must trust all of the following for *every* rule:

- the loader;
- the three-valued evaluator;
- the budget accounting;
- the template renderer;
- the IR semantics spec;
- a machine-emitted JCS pack with no comments and renumbered steps;
- the seven stages of `cdlc`, to confirm the pack derives from the CDL;
- the CDL source;
- the pinned spec.

This is interpreter trust, and the trusted base grows rather than shrinks.

**Step ordering is also at risk.** Step IDs are strings (`"3a"`, `"10"`, `"12a"`,
`"pre-flight"`). Any "canonical order" that sorts by ID puts "10" before "2". The design does
not say what tiebreaks the `after` partial order.

**Fix.** Keep the native walk (Finding 1). If a pack ever drives a step, the verifier must:

- print a step trace listing each executed step and each primitive call, counted by the
  *host* primitive binding rather than reported by the evaluator (this is also what makes
  N014's `not_reached "9"` a real check instead of self-report);
- carry step order as an explicit list in the IR, never derived by sorting.

### 9. The go:embed supply chain produces a self-attested hash. **Medium.** Attacks 00 R3, DSL-006.

**Failures.**

- **The hash vouches for itself.** `rulepack.sha256` is embedded next to `rulepack.json`,
  and the binary prints `rulepack_sha256` in the verdict. A tampered binary prints whatever
  hash it likes.
- **Pack substitution is unspecified.** The design does not say whether any CLI flag or
  environment variable can load another pack. If one can, an institution can hand an
  examiner a "verifier plus pack" pair.
- **The reproducibility check has a heavy base.** It needs the exact `cdlc` version, the Go
  toolchain and the pinned spec.
- **Spec §10.26 already requires more:** reproducible builds, Cosign-signed release
  artifacts, and an SBOM.

**Fix.**

- Publish the pack hash in the spec's §11 pin and in the signed release manifest.
- Allow no pack override on the conformant path. A test-only override must force a distinct
  status line (`Status: NONCONFORMANT-RULEPACK`) and exit 3.
- Document an offline rebuild recipe that yields both the binary hash and the pack hash.

### 10. The budget lets an examinee turn FAIL into "configuration error". **Medium.** Attacks 00 R6, 01 §4 Budget, 02 §6.

**Failures.**

- **Exhaustion reads as the examiner's problem.** Exceeding the budget is "a
  configuration-class outcome", which is exit 3 (01 §4). The examinee controls the input.
  A chain built to exhaust a relation's budget, for example many retail communications
  sharing one parent so the index join degenerates, yields an inconclusive exit 3 that
  reads as the examiner's misconfiguration, not a FAIL.
- **Budgets differ across runtimes.** "Evaluation steps" is not normatively defined, and
  the index plan's location is unclear: `cdlc` "plans", but does the IR carry the plan?
  Go and Python can therefore exhaust at different points, and the differential becomes
  noise.

**Fix.**

- Budget exhaustion is FAIL at the step that was running, with a normative reason. It is
  never exit 0 or exit 3.
- Put the cost model and the join plan in the IR, and make them normative.
- Measure cost identically in every runtime, and add budget cases to
  `conformance/runtime/`.

### 11. The primitive-binding surface is weakly typed and host semantics are unpinned. **Medium.** Attacks 01 §4 Primitive interface.

**Failures.**

- **Argument swaps are invisible.** Primitives bind by bare name. `hmac_sha256(key, msg)`
  with its two `bytes` arguments swapped type-checks.
- **Names are ambiguous.** `fingerprint` already has two constructions in Go (Research 01
  §4).
- **Behavior varies by library.** `ed25519_verify` behaves differently across libraries on
  non-canonical encodings and cofactored versus cofactorless verification. Unless the
  equation is pinned, the Go and Python witnesses disagree on edge signatures, or worse,
  agree by accident.
- **The self-test must not come from the pack.** The normative JCS self-test (spec §7,
  pre-flight) must be "compiled into the verifier binary". Its fixture must not come from
  the pack.

**Fix.**

- Keep a host-owned registry of versioned names (`hmac_sha256@1`) with nominal parameter
  types (`Key`, `Msg`).
- Run known-answer tests compiled into each host at bind time.
- Pin an Ed25519 verification profile in the IR semantics spec.
- The JCS self-test is native and runs before the pack loads.

### 12. "Independent" means one vendor. **Medium.** Attacks DSL-007, 00 Consumers, GOVERNANCE foundation criteria.

**Failure.** The ≥3 independent implementations that GOVERNANCE requires would be:

- a Go runtime, embedded in the vendor's verifier;
- a Python runtime, whose differential runs *inside* the proprietary TesseraSeal emit
  workflow;
- a .NET runtime, inside the license-gated Herald.Compliance (C4).

All three come from MMPWorks. All three consume a pack that MMPWorks authored, and the
authoring is "mostly through agents" (03 §5 risk 6). Agents given the same IR spec produce
correlated code. N-version-programming results (Knight and Leveson) show correlated failures
even across *separate* human teams. A foundation reviewer or an examiner will count this as
one implementation.

**Fix.**

- Record author and reviewer per rule and per vector, and require the expected-outcome
  reviewer to differ from the rule author.
- Count implementations by organization.
- Encourage third parties to implement **from the spec**, and treat the pack as optional
  reference data, never as a conformance requirement.

### 13. Closed-by-default families contradict the spec's forward-compatibility rules. **Low.** Attacks 00 R5 and 02 §4.

**Failure.** R5 and 02 §4 treat an undeclared field as a violation. The spec's
forward-compatibility rules (spec §7, items 2, 3 and 5) require the opposite:

- unknown wire-format kinds pass with an anomaly and `unknown_kind_present`;
- the MAC verifies regardless of unknown OTel attributes;
- unknown seal fields are tolerated.

The design conflates two different things: failing closed on an unknown *pack* construct,
which is correct, and failing closed on unknown *input*, which the spec forbids.

**Fix.** Keep the two notions separate in the IR semantics spec. For input, closed-ness is
per-family and explicit, and the §7 forward-compatibility dispatch is a sealed native
behavior.
