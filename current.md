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
- 2026-09-30: disclosure and contact wording fixed in the spec repo and the verifier repo,
  on branch `claude/busy-faraday-9sf328` in each. Not yet merged.

## Open questions
- The maintainer's local spec corpus (newer than GitHub) must be pushed before P0 can
  start.
- Patent and contribution-provenance questions are held privately.

## Next action
P0. Once the current spec corpus is public:
1. Re-run the rule inventory against it and refresh `docs/design/drift-register.md`.
2. Classify each drift item as `impl-bug` or `spec-meaning`.
3. Hand-fix D16, D18, the constant-time compares and the step-11 silent skip in `ffiec`.
4. Build the spec coverage matrix.

## Stop condition
- Do not write rules entries before the P0 re-survey. The public spec copy is stale.
- Never commit `docs/research/` or `docs/internal/`.
