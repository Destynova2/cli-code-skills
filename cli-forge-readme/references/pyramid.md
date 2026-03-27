# The 3-Tier Pyramid — Detailed Structure

> **When to read:** During Step 3 when writing the README using the pyramid structure.

---

## TIER 1 — The Hook (everyone reads this)

```markdown
# Project Name

> One-liner: what it does, for whom, in plain language.

[badges: CI status | version | license — max 4]

[Optional: 1 screenshot, GIF, or architecture diagram]
```

**Rules:**
- Title = project name, nothing else
- One-liner: zero jargon. A PM or a junior can understand it
- Badges: only actionable ones. No vanity badges
- Visual: terminal GIF (CLIs), diagram (infra/libs), screenshot (apps)

---

## TIER 2 — Get Started (most people stop here)

```markdown
## Quickstart

[3 commands max → visible result]

## What it does

[Feature list — short, scannable, max 5 items]

## Examples

[Real-world usage with expected output shown]
```

**Rules:**
- Quickstart must show a **visible result** — not just "it compiled"
- Show actual output: terminal for CLIs, curl response for APIs, `terraform output` for IaC
- Max 5 features listed. Link to docs for the rest
- Examples use realistic data, not `foo/bar/baz`

---

## TIER 3 — Contribute & Understand (devs / QA only)

```markdown
## Project Structure

[Simplified tree, max 2 levels, every folder annotated in ≤5 words]

## Development

### Prerequisites
### Build & Run
### Tests
### Linting / QA

## Architecture

[Only if non-obvious. Link to ADRs if they exist]

## Contributing

## Roadmap

## License
```

**Rules:**
- Project Structure: annotated `tree`, not a wall of paths
- Dev commands: copy-paste ready. Show what a passing test run looks like
- No dead sections: if you can't fill it, don't include it
- Contributing: link to CONTRIBUTING.md or 3–5 lines inline

---

## Useful Badges (copy-paste)

```markdown
[![CI](https://github.com/USER/REPO/actions/workflows/ci.yml/badge.svg)](https://github.com/USER/REPO/actions)
[![Crates.io](https://img.shields.io/crates/v/NAME)](https://crates.io/crates/NAME)
[![npm](https://img.shields.io/npm/v/NAME)](https://www.npmjs.com/package/NAME)
[![PyPI](https://img.shields.io/pypi/v/NAME)](https://pypi.org/project/NAME)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![License: AGPL-3.0](https://img.shields.io/badge/License-AGPL--3.0-blue.svg)](LICENSE)
[![MSRV](https://img.shields.io/badge/MSRV-1.XX-blue)](Cargo.toml)
[![Docs](https://docs.rs/NAME/badge.svg)](https://docs.rs/NAME)
```
