# Current State
_as of 2026-09-30_

## What we're building
Shared rules tooling for the FFIEC chain-of-custody verifier (Go, public) and the
conformance-vector toolchain. **Design v1 only; no code yet.**

## Active decisions
- 2026-09-30: v0 (a custom language with a compiler and interpreters) was rejected by
  red team. v1 (DSL-011) replaces it: typed data, generated constants, small table engines,
  and hand-written procedures.
- 2026-09-30 (Steve): everything is open source (Apache-2.0). The rules syntax is public so
  the community can improve it. The rules file lives in the spec repo. This repo is
  published from a clean history, with research kept out by `.gitignore`.
- 2026-09-30: disclosure and contact wording fixed in the canonical spec repo
  (mmpworks/SR-26.2-Model-Risk-Management PR #8) and the verifier (smuchow1962/ffiec PR #8).
  Neither is merged yet.
- The canonical spec repo is `mmpworks/SR-26.2-Model-Risk-Management` (local `ffiec-public`).
  `smuchow1962/ffiec-chain-of-custody` is its published submission output and is never
  hand-edited.

## Open questions
- The P0 re-survey targets `mmpworks/SR-26.2-Model-Risk-Management`, branch
  `spec/multi-payload-full-blob-hash-convention` (pushed 2026-09-30).
- Patent and contribution-provenance questions are held privately.

## Next action
P0. Against the canonical spec repo:
1. Re-run the rule inventory against it and refresh `docs/design/drift-register.md`.
2. Classify each drift item as `impl-bug` or `spec-meaning`.
3. Hand-fix D16, D18, the constant-time compares and the step-11 silent skip in `ffiec`.
4. Build the spec coverage matrix.

## Stop condition
- Do not write rules entries before the P0 re-survey.
- Never commit `docs/research/` or `docs/internal/`.
