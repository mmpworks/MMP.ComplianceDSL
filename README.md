# MMP.ComplianceDSL

Shared rules tooling for the **FFIEC AI chain-of-custody** proposed standard. It serves the
open-source reference verifier and the conformance-vector toolchain.

**Status: design v1 (2026-09-30). No code yet.** Licensed Apache-2.0.

## What it is

The chain-of-custody rules are currently copied by hand into the spec prose, a Go verifier,
a .NET implementation and a set of Python vector scripts, and the copies have drifted
(`docs/design/drift-register.md`). This project gives every implementation one source for
the parts of the spec that are data, while keeping every verification decision in
hand-written, independent code.

- **The rules format**: one closed JSON document shape for constants, enums, statuses and
  exit codes, §7 step IDs and reason templates, attribute-family schemas, and state
  machines. **The format is public so the community can improve it.**
- **The rules file** is an unofficial transcription of the spec, and the spec prose
  prevails on any conflict. It lives in the specification repository, under spec
  governance: https://github.com/smuchow1962/ffiec-chain-of-custody
- **`rulesgen`**: a Go program (standard library only) that validates the rules file and
  generates checked-in Go and Python code (C# later). A tripwire test forbids hand-held
  copies elsewhere.
- **Table engines**: a family checker and a state-machine walker, written by hand and
  independently in each language.
- **An open negative-vector manifest and reference materializer**, so anyone can regenerate
  the conformance corpus they are tested against.

The §7 verification walk, the cryptographic composition and the checks that link events
stay hand-written in each implementation. Only strings and tables are shared. The rule of
thumb: if getting it wrong could turn a FAIL into a PASS, it is code.

## Why it isn't a programming language

The first design (v0) proposed one, with a compiler and interpreters. Three red-team reviews
rejected it on assurance, engineering-cost and governance grounds. The two technical
reviews are in `docs/redteam/`, with a disposition for every finding in
`docs/redteam/dispositions.md`.

## Reading order

1. `docs/design/00-charter.md`: problem, goals, requirements.
2. `docs/design/01-architecture.md`: what is data and what is code.
3. `docs/design/02-rules-format.md`: the document shape and the engine semantics.
4. `docs/design/03-phases-and-verification.md`: phases P0–P5, gates, budget.
5. `docs/design/decisions.md` and `docs/design/drift-register.md`.
6. `docs/redteam/`: RT-1 (assurance), RT-2 (language and engineering), dispositions.

## Contributing and disclosure

See [CONTRIBUTING.md](CONTRIBUTING.md). The maintainer's company, MMPWorks LLC, develops
commercial software that implements the specification; the disclosure is in
CONTRIBUTING.md.
