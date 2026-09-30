# 01: Architecture (v1)

Status: **v1, after red team.** Supersedes the v0 design (unpublished; summarized in `decisions.md`).

## 1. Shape

```
  spec prose (pinned by spec-repo tag + commit, never a vendored copy)
        │  transcribed by hand, cited per entry
        ▼
  rules/chain-of-custody.rules.json      one closed JSON document (schema: spec/rules.schema.json)
        │
        │  rulesgen  (Go, stdlib only)
        │   load with DisallowUnknownFields → resolve refs → check templates
        │   → check safety invariants → refuse "disputed" entries → emit
        ▼
  ┌──────────────────┬────────────────────┬──────────────────┬────────────────────────┐
  │ gen/go/rules_gen │ gen/py/rules_gen.py│ gen/cs/Rules.g.cs│ gen/docs/*.md          │
  │ .go (stdlib only)│                    │ (P5)             │ tables, drift report,  │
  │                  │                    │                  │ coverage matrix        │
  └────────┬─────────┴─────────┬──────────┴────────┬─────────┴────────────────────────┘
           │ data only         │ data only         │ data only
  ┌────────▼─────────┐ ┌───────▼──────────┐ ┌──────▼───────────┐
  │ Go table engines │ │ Py table engines │ │ .NET (existing   │  hand-written,
  │ family checker   │ │ family checker   │ │ StateMachine.cs) │  independent
  │ machine walker   │ │ machine walker   │ │                  │
  └────────┬─────────┘ └───────┬──────────┘ └──────┬───────────┘
  ┌────────▼─────────┐ ┌───────▼──────────┐ ┌──────▼───────────┐
  │ ffiec §7 walk,   │ │ reference        │ │ Herald.Compliance│  hand-written,
  │ relations, crypto│ │ materializer +   │ │ ChainVerifier    │  independent;
  │ composition (Go) │ │ Py witness walk  │ │                  │  logic never in data
  └──────────────────┘ └───────┬──────────┘ └──────────────────┘
                               │
                    commercial tools (e.g. MMPWorks TesseraSeal) may build corpus
                    expansion ON the open manifest + materializer
```

## 2. What is data and what is code

This table is the core of the design.

| Data (in `rules.json`; generated into every language) | Code (hand-written in each language) |
|---|---|
| Constants: HKDF salt and info, genesis, Merkle prefixes, lengths | HKDF, HMAC, SHA-256, RFC 6962, Ed25519, JCS (native, audited) |
| Enums with wire strings; algorithm-string table | **Which bytes are MACed, hashed or signed** (crypto composition) |
| Outcome statuses → exit codes (checked against invariants) | Constant-time comparison (`subtle.ConstantTimeCompare`, `hmac.compare_digest`) |
| Step IDs, **integer ordinals**, owning section, reason templates (pre-split, typed slots, explicit `quoted`) | The §7 walk: order, short-circuit, which steps run, modes, strict |
| Family schemas: fields, closed type set, required, enum refs, forward-compatibility policy | Relations between events (§10.84, adjuster bidirectional, retrieval, re-seal) |
| Conditional presence rows: `when field ∈ {…} require [fields]` (fixed shape, no expressions) | Guards that don't fit a row: map-key membership, ranges, "eventually resolved" |
| State machines: states, start, transitions, terminal, key fields, **ordered** checks with reasons | Backfill, bordereau per-party guards, dual-algorithm cases (a)–(e) |
| Negative manifest schema (separate file, `spec/negative-manifest.schema.json`) | Materialization (open reference, Python); commercial expansion layers are optional |

**The test for any new item:** if getting it wrong could turn a FAIL into a PASS, it is
code, written by hand in each language. If getting it wrong would only produce a wrong
*string* or a wrong *table row*, and the host invariants plus the corpus would catch it,
it is data.

## 3. Components

| Path | Language | License | Role |
|---|---|---|---|
| `spec/rules.schema.json` + `spec/rules-format.md` | JSON Schema + prose | Apache | Shape and meaning of `rules.json`. Short. |
| `spec/negative-manifest.schema.json` | JSON Schema | Apache | Negative vector declarations: base vector, layer (`pre_chain` / `bytes` / `context`), parameters, expected Status/Step/Reason/Exit, `not_reached` (step, seq). |
| `spec/input-profile.md` | prose | Apache | The I-JSON parsing profile (R6) plus its conformance cases. |
| `rulesgen/` | Go, stdlib only | Apache | Validator and generator. About 1k lines plus tests. |
| `engines/go/` | Go, stdlib only | Apache | Family checker and machine walker. About 500 lines. ffiec vendors it. |
| `engines/py/` | Python | Apache | The same, written independently. |
| `materializer/` | Python | Apache | Reference materializer: manifest + base vector → fixture files. Uses one primitive module (JCS via `jcs`, RFC 6962 with a single split algorithm, HKDF with tenant and device info, sign_payload versions) that replaces the 25 scripts' copies. |
| `conformance/` | JSON | Apache | Engine conformance cases and input-profile cases. |
| `rules/chain-of-custody.rules.json` | JSON | Apache | The transcription. **It lives in the spec repo** (`smuchow1962/ffiec-chain-of-custody`), under spec governance, not here (DSL-012). |

**Not in this repo:**

- ffiec's §7 walk and relations stay in `ffiec`.
- TesseraSeal's expansion and review workflow stay in the VectorCompiler.
- Herald.Compliance procedures stay in .NET.

## 4. How consumers use it

**ffiec (Go).**

- It imports the generated `rules_gen.go` and `engines/go` as a vendored module. An
  examiner can build it offline.
- `auditwalk.go` stays a short list of hand-written steps. Its strings come from the
  generated data.
- The family predicates move to the family checker driven by the tables.
- `communication_preapproval.go` stays hand-written.
- A tripwire test fails the build on any string literal that matches a known reason prefix
  outside `rules_gen.go`.

**TesseraSeal (Python).**

- Task #24 is rebuilt on the open negative manifest, the materializer and the Python
  engines.
- The proprietary value moves to three places:
  - generating manifests at scale (parameterized families, mutation sweeps);
  - corruption depth beyond the reference layer set;
  - the emit, review and attestation workflow.
- `FailureClass` is deleted. Expected outcomes are the manifest's triple.

**Herald.Compliance (.NET, P5).**

- `Rules.g.cs` replaces the enums and transition tables in `ClaimState.cs`,
  `Bordereau.cs` and the others.
- The 5000-series error codes map onto step IDs through the table.

## 5. Versioning

- `rules.json` carries:
  - `format_version` (the shape);
  - `spec_ref` (the spec repo URL, tag and commit);
  - `transcription_version`.
- Generated files carry all three plus `rules_sha256`.
- A host refuses generated data whose `format_version` it doesn't support.
- One `rules.json` means one spec version.

## 6. Security posture of the data

The rules data is a build input, not a runtime input.

- **It is compiled in**, not loaded at runtime. The verifier accepts no `--rules` flag, so
  an examinee cannot swap it.
- **Host invariants (R7) run twice:** in `rulesgen` at generation time, and as a unit test
  in each host.
- **Resource budgets** are properties of the hand-written walk. A budget overrun is a
  distinct `STRUCTURAL` outcome, with its reason naming the input size. It is never a
  configuration error an examinee could induce to turn a FAIL into "inconclusive"
  (RT-1 F10). The caps are fixed constants, not scaled to the input.
