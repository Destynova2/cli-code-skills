---
name: cli-forge-doc
description: Generate and audit comprehensive project documentation from a Git repository. Produces dual output — an LLM-optimized context file (AGENTS.md, llms.txt) and a human-readable documentation site (Diátaxis structure). Use this skill whenever someone asks to document a project, generate docs, create a README, write API docs, create an AGENTS.md or CLAUDE.md, generate llms.txt, or improve existing documentation. Also trigger when someone mentions "doc", "documentation", "readme", "explain this codebase", "onboard developers", or "make this project understandable".
argument-hint: "[git-repo-path]"
context: fork
agent: general-purpose
---

> **Language rule:** Detect the project's primary language (from README, comments, docs, commit messages). Output your documentation in that language. If the project is bilingual, ask the user which language to use before proceeding.

# Doc Forge — Dual Documentation Generator & Auditor

Generate production-grade documentation for any Git project, optimized for both AI agents and human readers.

## Philosophy

Three principles stolen from the best:

1. **OpenBSD doctrine**: "Developers making a change to the system are expected to update the man pages along with their change." Docs ship with code. Man pages are authoritative. If it's not documented, it doesn't exist.
2. **Diátaxis framework**: Documentation has four modes — tutorials, how-to guides, reference, explanation. Never mix them.
3. **Dual-audience reality**: In 2025+, your docs serve two audiences — humans reading progressively, and LLMs consuming flat context. Serve both.

## Input

`$ARGUMENTS` is the path to the Git repository (or current directory if empty).

If no argument: look for a Git repo in the current working directory.

## Step 0: Reconnaissance

Before writing a single line of doc, **read the project**. This is non-negotiable.

```
1. List the root directory
2. Read: README.md, CLAUDE.md, AGENTS.md, Cargo.toml, package.json, pyproject.toml, go.mod, Makefile, Dockerfile, docker-compose.yml, .gitlab-ci.yml, .github/workflows/*, justfile
3. Identify: primary language(s), build system, CI/CD, test framework, deployment method
4. Sample 10-15 key source files (entry points, public API, core modules)
5. Read: docs/ directory if it exists
6. Check: git log --oneline -20 for recent activity focus
7. Check: git remote -v for repo URL
```

Build a mental model of:
- **What** the project does (purpose, problem it solves)
- **How** it's structured (architecture, key abstractions)
- **Who** uses it (library consumers, CLI users, operators, contributors)
- **Where** it runs (local, cloud, air-gapped, container)

## Step 1: Completeness Scan (The Checklist)

Before generating, score what exists. Use the **Documentation Completeness Index (DCI)**.

### DCI Formula

```
DCI = Σ(wᵢ × sᵢ) / Σ(wᵢ) × 10

Where:
  wᵢ = weight of item i (importance)
  sᵢ = score of item i (0.0 to 1.0)
```

### Checklist Items (with weights)

| # | Item | Weight | What to check |
|---|------|--------|---------------|
| 1 | Project overview (README or equivalent) | 5 | Exists, has purpose statement, not just boilerplate |
| 2 | Getting started / quickstart | 5 | Can a new dev go from clone to running in <10 min? |
| 3 | Architecture overview | 4 | High-level structure documented somewhere |
| 4 | API reference (public surface) | 5 | Doc comments on public items, generated docs |
| 5 | Configuration reference | 3 | All config options documented with defaults |
| 6 | Error handling guide | 3 | Error types, what they mean, how to fix |
| 7 | Deployment / operations guide | 3 | How to deploy, monitor, troubleshoot |
| 8 | Contributing guide | 2 | How to contribute, code style, PR process |
| 9 | Changelog / release notes | 2 | History of changes |
| 10 | License | 1 | Exists and is clear |
| 11 | CI/CD documentation | 2 | Pipeline stages, environments, secrets needed |
| 12 | Security documentation | 3 | Threat model, auth flows, secrets management |
| 13 | LLM context file (AGENTS.md / CLAUDE.md) | 3 | AI-readable project context |
| 14 | Examples / tutorials | 4 | Working, tested, progressive complexity |
| 15 | Inline doc coverage (public API) | 4 | % of public items with doc comments |
| 16 | Cross-references & linking | 2 | Types linked, related items connected |

**Scoring guide**: 0.0 = absent, 0.25 = stub/placeholder, 0.5 = exists but incomplete, 0.75 = good but gaps, 1.0 = excellent.

Output the DCI score table and overall score before proceeding.

## Step 2: Generate LLM Documentation (Machine Layer)

This layer makes the project consumable by AI coding agents. It follows the approach of Microsoft's Rust Guidelines `all.txt` and the `llms.txt` standard.

### 2A: AGENTS.md (Project Context for AI Agents)

Generate a single flat file that gives an AI agent everything it needs to work on this project. Keep it **under 300 lines** (respect the instruction budget).

**Structure:**

```markdown
# [Project Name]

[One sentence: what it does and why it exists.]

## Stack
[Language, framework, key deps — one line each]

## Architecture
[3-5 sentences describing how the pieces fit together. Describe capabilities, not file paths.]

## Domain Concepts
[Glossary of project-specific terms the agent must understand]

## Key Patterns
[Non-obvious conventions: error handling style, naming conventions, module organization]

## Commands
[Exact commands for: build, test, lint, run, deploy — copy-pasteable]

## Gotchas
[Things that will trip up an agent: non-obvious env vars, quirky configs, known footguns]
```

**Rules for AGENTS.md** (from research synthesis):
- Describe capabilities, not file paths (paths change, capabilities don't)
- Domain concepts are stable → document them. File structure is volatile → don't.
- One sentence project description acts as a role-based prompt for the agent
- Include package manager explicitly if non-default
- Never dump style guides here — let the linter do the linter's job
- Use progressive disclosure: point to `docs/` for details, don't inline everything

### 2B: llms.txt (Documentation Index for LLMs)

Generate a `llms.txt` following the llmstxt.org standard:

```markdown
# [Project Name]

> [One paragraph summary]

## Documentation
- [Getting Started](docs/getting-started.md): Setup and first steps
- [Architecture](docs/explanation/architecture.md): System design overview
- [API Reference](docs/reference/api.md): Complete API documentation

## Source
- [Main entry](src/main.rs): Application entry point
- [Core module](src/core/mod.rs): Core business logic

## Configuration
- [Config reference](docs/reference/config.md): All configuration options
```

### 2C: llms-full.txt (Optional, for small projects)

If the project's total doc content fits under ~50K tokens, generate a single concatenated file with all documentation content inline, eliminating the need for link traversal.

## Step 3: Generate Human Documentation (Diátaxis Layer)

This layer serves humans — from newcomers to experts. Follow the **Diátaxis framework** strictly: tutorials, how-to guides, reference, explanation. Never mix modes.

### Directory Structure

```
docs/
├── index.md                    # Landing page — what is this? who is it for?
├── tutorials/
│   ├── getting-started.md      # First contact — clone to running
│   └── first-feature.md        # Build something meaningful
├── how-to/
│   ├── configure.md            # How to configure [common scenario]
│   ├── deploy.md               # How to deploy to [target]
│   ├── troubleshoot.md         # Common problems and solutions
│   └── contribute.md           # How to contribute
├── reference/
│   ├── api.md                  # API surface (or generated from code)
│   ├── config.md               # All config options, defaults, types
│   ├── cli.md                  # CLI commands and flags
│   └── errors.md               # Error codes and meanings
├── explanation/
│   ├── architecture.md         # Why it's built this way
│   ├── decisions/              # ADRs (Architecture Decision Records)
│   │   └── 001-why-rust.md
│   └── security.md             # Security model and threat assumptions
├── design/                        # Design docs (thinking tools, pre-implementation)
│   └── 000-template.md            # Design doc template
├── CHANGELOG.md
└── CONTRIBUTING.md
```

### Writing Rules per Diátaxis Quadrant

**Tutorials** (learning-oriented):
- Take the reader by the hand. Every step must work.
- Start with the simplest possible success, then build up.
- Don't explain why — just show what. Link to explanations.
- Test every tutorial: can a fresh dev follow it?
- The reader should feel accomplishment at each checkpoint.

**How-to Guides** (task-oriented):
- Title = "How to [verb] [object]"
- Assume competence. Don't explain basics.
- Focus on the goal, not the tool.
- Provide forkable paths for variants.
- Link to reference for exhaustive options.

**Reference** (information-oriented):
- Structure mirrors the code: if the API has 4 modules, reference has 4 sections.
- Exhaustive and factual. No opinions, no tutorials.
- Every config option, every error code, every CLI flag.
- Generated from code when possible (rustdoc, typedoc, sphinx).

**Explanation** (understanding-oriented):
- Answer "why" questions. Why this architecture? Why not the alternative?
- ADRs (Architecture Decision Records) live here.
- Free to compare, digress, tell history.
- This is where you document the thinking, not the doing.

**Design Docs** (thinking tool — pre-implementation):

Design docs are NOT post-hoc documentation. They are a **thinking process** that happens before code is written. The document itself is the design work. (Inspired by Google's engineering design doc practice and Ousterhout's "A Philosophy of Software Design".)

Structure:
```markdown
# Design Doc: [Title]

**Author**: [name]
**Status**: Draft | In Review | Approved | Superseded
**Date**: [date]
**Reviewers**: [names]

## Context
[What situation are we in? What problem exists? What triggered this work?]

## Goals
[What must this solution achieve? Be specific and measurable.]

## Non-Goals
[What is explicitly OUT OF SCOPE? This is the most valuable section —
it prevents scope creep and clarifies thinking.]

## Proposed Design
[The solution. Describe it at the right level of abstraction —
enough to evaluate tradeoffs, not so much it's pseudo-code.]

## Alternatives Considered
[For each alternative:
- What is it?
- Why is it a reasonable option?
- Why did we NOT choose it?
The rejected alternatives are as important as the chosen one.]

## Risks & Mitigations
[What could go wrong? How do we detect it? How do we recover?]

## Open Questions
[What don't we know yet? What needs more investigation?
Honest uncertainty is better than false confidence.]
```

Rules for design docs:
- Write BEFORE coding. If the design doc comes after the code, it's documentation, not design.
- The **Non-Goals** section is where most of the thinking happens. If you can't articulate what you're NOT doing, you don't understand what you ARE doing.
- **Alternatives Considered** must have at least 2 genuine alternatives (not strawmen). If there's only one option, you haven't explored the space.
- Keep it short. A design doc that nobody reads is worse than none. Target 1-3 pages.
- Design docs live in `docs/design/` and are never deleted — they're historical records of how decisions were made.

### Progressive Complexity

Human docs should follow a **funnel**:

```
Level 0: "What is this?" → docs/index.md (30 seconds)
Level 1: "Let me try it" → tutorials/getting-started.md (10 minutes)
Level 2: "I need to do X" → how-to/ (task-specific)
Level 3: "What are all the options?" → reference/ (exhaustive)
Level 4: "Why is it built this way?" → explanation/ (deep understanding)
```

Each level should link naturally to the next. A reader should never feel lost or need to ask "where do I go from here?"

## Step 4: Audit (Score the Result)

After generation, run the doc-audit framework on the result. See `reference.md` for the full audit methodology.

### Quick Audit Checklist

For each generated file, verify:

1. **Accuracy**: Does the doc match the actual code? (Read the implementation.)
2. **Completeness**: Are all public items covered?
3. **Freshness**: Does it reference current APIs, not stale ones?
4. **Examples**: Do examples compile/run? Do they use `?` not `unwrap()`?
5. **Cross-refs**: Are types and functions linked, not just named?
6. **Progressive**: Can a newcomer navigate from zero to competent?

### Documentation Debt Score

```
Doc Debt = (Public Items − Documented Items) / Public Items × 100

Thresholds:
  0-10%  → Green  (healthy)
  10-25% → Yellow (needs attention)
  25-50% → Orange (significant debt)
  50%+   → Red    (documentation emergency)
```

## Step 5: Output

### Files to Generate

At minimum, produce:
1. `AGENTS.md` — LLM context file (root of repo)
2. `docs/index.md` — Human landing page
3. `docs/tutorials/getting-started.md` — First-contact tutorial
4. `DCI-REPORT.md` — Documentation completeness index with scores and recommendations

If the project warrants it (public API, library, complex system), also produce:
- `llms.txt` / `llms-full.txt`
- Full Diátaxis docs/ tree
- Updated inline doc comments (as a diff or suggestions)
- `docs/design/000-template.md` — Design doc template for the team to reuse
- Design docs for major existing architectural decisions (if none exist yet)

### Presentation

Present the DCI score first, then the generated files. Summarize:
- Current state (DCI score)
- What was generated
- Top 3 highest-impact improvements still needed
- Recommended next steps (ordered by effort vs impact)

## Adaptation Rules

1. **Scale to project size**: A 200-line CLI script needs a good README, not a docs site. A platform with 50K LOC needs the full Diátaxis treatment.
2. **Match the language**: Follow the language's idiomatic doc conventions (see `reference.md`).
3. **Respect existing docs**: Don't overwrite good existing docs. Augment and cross-reference.
4. **Monorepo support**: If the project has multiple packages/services, generate per-package docs + a root-level overview.
5. **Air-gapped aware**: If the project targets air-gapped environments, emphasize offline-complete documentation. No external links that won't resolve.
6. **Confidence markers**: If you're unsure about something (inferred from code but not confirmed), mark it with `<!-- NEEDS-REVIEW: [reason] -->` rather than guessing.
