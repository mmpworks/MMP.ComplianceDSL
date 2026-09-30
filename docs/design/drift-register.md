# Drift register

Status: seed list, 2026-09-30. It was compiled against spec `0.1.0-draft.7` and must be re-run
against the current spec corpus in P0. Each item will be classified `impl-bug` or
`spec-meaning` (DSL-020). Every `spec-meaning` item becomes a public spec proposal.

This is the seed list: things that **must** be decided once, because a rule pack cannot
hold two answers.

| # | Drift | Sides |
|---|---|---|
| D1 | Exit-code meanings | spec 1=FAIL, 2=structural, 3=config, vs .NET and vector 022 |
| D2 | `format_version` quoting | N022 and .NET unquoted, vs N023 and Go quoted |
| D3 | .NET cannot reach step 1 for N022/N023 | .NET only |
| D4 | Step 3a | missing in Go; in the wrong position in .NET |
| D5 | Step 4 run_id check | missing in Go |
| D6 | Step 6 check order and which seq is reported | Go vs .NET |
| D7 | Reason suffixes | .NET's break the `: ` extension rule |
| D8 | sign_payload | Go "running v1.0c" vs spec and .NET "v1.0b" |
| D9 | Backfill (N025) | three tuple shapes and three reason wordings |
| D10 | N033 reason | spec wording vs Go's base-walk classification |
| D11 | Streaming unknown kind | Go ignores it; .NET and vector 022 fail with exit 1 |
| D12 | Bordereau | .NET KeyNotFound bug; guards (b) and (c) missing |
| D13 | Claim table self-loops | vector 037 vs 038 and .NET |
| D14 | Spec CHANGELOG exit codes 12 and 14 vs §7/§10.12 | "no new exit codes" |
| D15 | Spec internal contradictions | routing five vs six; pre-amendment 6 vs 7 lines; truncation exit code |
| D16 | Go has two chain shapes; the CLI runs the non-conformant one | Go only |
| D17 | Go-only families use flat dotted keys mixed with nested objects | Go only |
| D18 | JCS key sort by UTF-8 instead of UTF-16 | Go `core/jcs` |
