# Decision log (v1)

Status key:

- **accepted**: settled after the red team.
- **steve**: needs the maintainer's decision.
- **superseded**: replaced by a later decision.
- **gated**: waiting on evidence.

## Superseded v0 decisions

The v0 design documents are not published. The rows below record what each one proposed.

| ID | v0 decision | Status | Replaced by / reason |
|---|---|---|---|
| DSL-001 | Share data (CDL, IR, rule pack), run it in N interpreters | superseded | DSL-011. One transcription run by N interpreters is one witness (RT-1 #1, RT-2 F1). |
| DSL-003 | A purpose-built textual language | superseded | DSL-011, DSL-016. See the re-evaluation below. |
| DSL-004 | Compiler in Go | superseded → kept in narrower form | DSL-013. The generator is in Go, standard library only. |
| DSL-005 | Crypto stays native, and data wires it together | superseded | DSL-014. Wiring crypto together in data is the category error (RT-1 #2). |
| DSL-006 | Vendor the runtime and embed the pack | superseded | DSL-015. The data is compiled in and cannot be loaded at runtime. |
| DSL-007 | Author declares, runtimes agree | kept, narrowed | Declaring the expected outcome catches engine bugs, not misreadings of the spec. The coverage matrix and outside implementers are what address misreadings. |
| DSL-008 | Three corruption layers | **accepted** | Moves into the open negative manifest. |
| DSL-009 | Integer-only IR | superseded | Exact decimal strings for ranges. There is no IR any more. |
| DSL-010 | One spec version per unit | **accepted** | One `rules.json` per spec version. |

## v1 decisions

| ID | Decision | Status |
|---|---|---|
| DSL-011 | **Typed data, generated constants, small table engines, procedures written by hand.** Data holds strings and tables. Code holds every decision on the examiner path. | accepted |
| DSL-012 | **Everything here is open source (Apache-2.0), including the rules format.** The format is public so the community can improve it. The rules file itself lives in the spec repo, under spec governance; this repo holds the format, the generator and the engines. | accepted 2026-09-30 |
| DSL-013 | `rulesgen` is Go, standard library only. It emits checked-in Go and Python (C# later). No runtime loading. | accepted |
| DSL-014 | Crypto primitives *and how they are wired together* stay hand-written in each host. So do constant-time comparisons. | accepted |
| DSL-015 | Rules data is compiled in. The verifier has no `--rules` flag. `rules_sha256` is printed on a diagnostic line after `Verdict-Object:`. | accepted |
| DSL-016 | Escape hatch: a closed JSON predicate AST only if more than about 40 cross-field conditions remain after P2. Never a grammar. | gated |
| DSL-017 | One undefined rule: *cannot evaluate* takes the severity of its rule. No vacuous `exists`. | accepted |
| DSL-018 | Parse inputs with the I-JSON profile: no duplicate keys, no integers above 2^53, case-exact keys, strict UTF-8. | accepted |
| DSL-019 | Hosts hard-code safety invariants. The generator also checks them. | accepted |
| DSL-020 | Drift is classified `impl-bug` or `spec-meaning`. A `spec-meaning` item becomes a public spec proposal, and the entry is not emitted until the proposal settles. | accepted |
| DSL-021 | An open negative manifest schema plus an open reference materializer. Commercial tools (including MMPWorks' TesseraSeal) may build on them; the open corpus never depends on them. | accepted |
| DSL-022 | Implementations written by MMPWorks are described as *reference implementations*, not independent ones. | accepted |
| DSL-023 | Don't call it an "FFIEC rule pack." The document declares itself an unofficial transcription in which the prose prevails. | accepted |

## DSL-003 re-evaluated

RT-2 F3 corrected three of v0's factual claims:

- **cel-go.** Its dependencies are as v0 stated: ANTLR, protobuf, genproto, and others.
  That still fails C1.
- **cel-python.** It exists, so "no independent Python CEL" was **wrong**.
- **Starlark.** v0 never considered it. It is pure Go but depends on `golang.org/x/sys`,
  which fails C1. It is also a general-purpose language, which v1 does not want on the
  examiner path.

v0 also compared against the wrong baseline. The comparison that matters is "JSON tables
plus hand-written code," and that alternative wins on cost, on how independent the
implementations stay, and on how easy the verifier is for an examiner to audit.
