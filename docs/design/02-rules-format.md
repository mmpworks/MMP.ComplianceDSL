# 02: The rules format (v1)

Status: **v1, after red team.** Supersedes the v0 design (unpublished; summarized in `decisions.md`). The examples below are
shapes only. Real entries are transcribed only after P0 re-surveys the current spec.

## 1. Document

```json
{
  "format_version": "1",
  "spec_ref": {"repo": "…/ffiec-chain-of-custody", "tag": "…", "commit": "…"},
  "transcription_version": "0.1.0",
  "unofficial": "Machine-readable transcription. The specification prose prevails on any conflict.",
  "constants": [ … ], "enums": [ … ], "statuses": [ … ], "steps": [ … ],
  "families": [ … ], "machines": [ … ], "drift": [ … ]
}
```

- Keys are closed. `rulesgen` loads the document with `DisallowUnknownFields`, and it also
  checks for duplicate keys with a token-stream pass.
- Every entry has an `id`, a `spec` (section) and a `quote_sha256`, and may have a
  `status` of `transcribed`, `disputed` or `unimplemented`.

## 2. Entries

**Constant**

```json
{"id":"HKDF_SALT","type":"bytes","utf8":"ffiec.chain-of-custody.v1.salt","spec":"§4.1","quote_sha256":"…"}
```

`type` is one of `bytes`, `int`, `string`. A `bytes` value is given as exactly one of
`utf8`, `hex` or `zeros`.

**Enum**

```json
{"id":"SignatureAlgorithm","spec":"§4.3","members":[{"id":"ed25519","wire":"ed25519"},{"id":"hmac_sha_256","wire":"HMAC-SHA-256"}],
 "open":null}
```

When an enum has the institution-named escape hatch, `open` is set to
`{"escape":"institution_named","obligation":"§10.12"}`.

**Status**

```json
{"id":"FAIL","wire":"FAIL","exit":1}
```

Invariant R7: FAIL must have a non-zero `exit`.

**Step**

```json
{"id":"8","ordinal":90,"spec":"§7","scope":"entry","exit_on_fail":1,
 "reason":{"parts":[{"lit":"key_fingerprint mismatch at seq "},{"slot":"seq","fmt":"int"},
                    {"lit":": looked-up IKM does not match the entry's recorded fingerprint"}]}}
```

- `ordinal` is an integer. `"3a"` sits between `"3"` and `"4"`. String IDs are never
  sorted.
- `scope` is `file`, `header`, `entry` or `day`.
- A slot's `fmt` is one of:
  - `int`;
  - `raw`: bytes as given;
  - `quoted`: JSON string quoting;
  - `hex`;
  - `date`.
- **The steps table says what each step is called, what it reports and when it runs. It
  never says what the step checks.** That is hand-written in each host.

**Family**

```json
{"id":"audit.challenge_response","spec":"§10.55","key_style":"nested",
 "fields":[{"name":"outcome","type":{"enum":"ChallengeOutcome"},"required":true},
           {"name":"modified_decision_sha256","type":"hex64","required":false}],
 "conditional":[{"when":{"field":"outcome","in":["modified"]},"require":["modified_decision_sha256"]},
                {"when":{"field":"outcome","not_in":["modified"]},"forbid":["modified_decision_sha256"]}],
 "unknown_fields":"forward_compat_anomaly"}
```

- **Field types come from a closed set:**
  - `string`
  - `int`
  - `bool`
  - `hex64`
  - `ts_utc_s`
  - `ts_utc_us`
  - `{"enum": ref}`
  - `{"list": T}`
  - `{"record": ref}`
  - `{"decimal": {"min": "0", "max": "1"}}`, compared as exact decimal strings, never as
    floats.
- **`key_style` is `flat` or `nested`.** It records the flat-dotted-key vs nested-object
  distinction (D17) instead of hiding it.
- **`unknown_fields` follows the spec's forward-compatibility rules:** an anomaly, or
  ignore, per the section. It is never a silent pass and never a blanket FAIL (RT-1 F13).
- **Conditional rows have one fixed shape:** `when` + `in` / `not_in` + `require` /
  `forbid`. There are no expressions.

**Machine**

```json
{"id":"ClaimLifecycle","spec":"§10.43","family":"audit.claim_state","key":["claim_id"],
 "from_field":"from_state","to_field":"to_state","start":"opened","terminal":["closed"],
 "transitions":{"opened":["opened","pending","closed"],"pending":["pending","decided","closed"],"decided":["decided","closed"],"closed":[]},
 "checks":[{"check":"history_gap","reason":{…}},{"check":"transition","reason":{…}},{"check":"start","reason":{…}}],
 "eventually_terminal":false}
```

- Checks run in array order, because the spec pins which reason wins.
- Walk order is by `(key, seq)`.
- An empty walk is reported explicitly, using the reason the spec gives for it.

**Drift**

```json
{"id":"D2","kind":"spec-meaning","title":"format_version quoting","sides":["N022 unquoted","N023 quoted"],
 "proposal":"<spec-proposal issue URL>","blocks":["steps/1.reason"]}
```

- For a `spec-meaning` drift, every entry it blocks is `disputed`, and the generator
  refuses to emit it.
- For an `impl-bug` drift, the record names the implementation and the fixing commit.

## 3. Engine semantics (normative for every hand-written engine)

**Family checker.** Input: one entry's attribute object (parsed under the input profile)
and a family entry.

1. **Presence and type.** A required field that is missing gives a FAIL-severity finding.
   A wrong type does too.
2. **Enum membership.** Compare wire strings, case-exactly.
3. **Conditional rows.** Evaluate the rows in order.
   - If the `when` field is absent, the row does **not** apply. The presence rules decide
     whether the field is required.
   - If the `when` field is present but has the wrong type, the result is a *cannot
     evaluate* finding at FAIL severity (R5).
4. **Unknown fields.** Apply the `unknown_fields` policy.

Output: an ordered list of findings, each `(family, field, rule_id, severity)`. The step
that runs the checker renders the finding's reason using the templates.

**Machine walker.** Input: the entries for one family.

1. Group the entries by `key`, and sort each group by `seq`.
2. For each group, apply `checks` in their declared order. The first failure per group
   wins.
3. If `eventually_terminal` is set, also check that each group reaches a terminal state.

Output: a list of findings, one per failing group.

**Conformance.** Every rule above has cases in `conformance/`. The Go and Python engines
must produce identical finding lists. If they disagree, one of them has a bug, not "a
different interpretation."

## 4. The escape hatch (DSL-016)

After P2, count the cross-field conditions that remain hand-written because rows cannot
hold them. If that count exceeds about 40, add a closed JSON predicate AST in the
PredicateSpec style:

- node kinds: `eq`, `in`, `present`, `and`, `or`, `not`;
- no binders and no quantifiers;
- evaluated by about 300 lines in each language.

There will never be a grammar.
