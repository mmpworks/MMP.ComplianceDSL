# Contributing

The rules format is public on purpose: community review is how it gets better.

## What to contribute

- **The rules format** (`spec/rules.schema.json`, `spec/rules-format.md`, once they land).
  Open an issue that names the spec rule the current format cannot express, or expresses
  badly. Show a concrete example. Changes to the format are versioned (`format_version`).
- **The generator and the table engines.** Bug reports need a minimal reproduction. A
  Go/Python disagreement on a conformance case is always a bug in one of them.
- **Design review.** Comments on `docs/design/` and `docs/redteam/` are welcome as issues.

## What does not belong here

- **The rules themselves.** The transcription (`rules/chain-of-custody.rules.json`) lives
  in the specification repository, under its governance:
  https://github.com/smuchow1962/ffiec-chain-of-custody
- **Questions about what the specification means.** File those as spec proposals in the
  specification repository. This repo never settles them (decision DSL-020).

## Terms

By contributing, you agree that your contribution is licensed under the Apache License
2.0, as set out in [LICENSE](LICENSE). No CLA is required; inbound equals outbound.

## Security

Report vulnerabilities privately through GitHub's private vulnerability reporting for this
repository (the **Security** tab, then **Report a vulnerability**). Do not open a public
issue.

## Disclosure

The maintainer, Steve Muchow, is the founder of MMPWorks LLC. MMPWorks develops commercial
software that implements the chain-of-custody specification, including TesseraSeal and
Herald.Compliance. Nothing in this repository depends on those products.
