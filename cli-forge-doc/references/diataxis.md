# Diataxis Framework — Templates & Rules

> **When to read:** Step 3 (Generate Human Documentation).

---

## Directory Structure

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

## Writing Rules per Quadrant

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

## Design Docs (thinking tool — pre-implementation)

Design docs are NOT post-hoc documentation. They are a **thinking process** that happens before code is written. (Inspired by Google's engineering design doc practice and Ousterhout's "A Philosophy of Software Design".)

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

## Progressive Complexity Funnel

```
Level 0: "What is this?" → docs/index.md (30 seconds)
Level 1: "Let me try it" → tutorials/getting-started.md (10 minutes)
Level 2: "I need to do X" → how-to/ (task-specific)
Level 3: "What are all the options?" → reference/ (exhaustive)
Level 4: "Why is it built this way?" → explanation/ (deep understanding)
```

Each level should link naturally to the next. A reader should never feel lost or need to ask "where do I go from here?"
