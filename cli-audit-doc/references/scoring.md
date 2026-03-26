# Documentation Quality Index (DQI) — Scoring Framework

> **When to read:** Step 4 (Compute final score).

---

## DQI Formula

```
DQI = Σ(wᵢ × sᵢ) / Σ(wᵢ) × 10

Where:
  wᵢ = weight of category i
  sᵢ = score of category i (0.0 to 1.0)
```

## Scoring Guide

| Score | Meaning |
|-------|---------|
| 0.0 | Absent — no documentation for this aspect |
| 0.25 | Stub — exists but placeholder quality |
| 0.5 | Present but incomplete — significant gaps |
| 0.75 | Good — covers most aspects, minor issues |
| 1.0 | Excellent — exemplary, could serve as a model |

## Category Weights

| # | Category | Weight | Rationale |
|---|----------|--------|-----------|
| C1 | Diataxis Coverage | 10% | All 4 doc types should be represented |
| C2 | Completeness | 12% | Public API coverage is the foundation |
| C3 | Freshness & Accuracy | 10% | Stale docs are worse than no docs |
| C4 | Readability & Prose Quality | 10% | Condescension and jargon repel readers |
| C5 | Structure & Findability | 8% | Unfindable docs are useless docs |
| C6 | Standard Sections | 8% | Errors, params, returns, examples |
| C7 | Code Examples | 10% | Working examples are the #1 learning tool |
| C8 | Accessibility & Inclusivity | 6% | Alt text, bias-free language, global readiness |
| C9 | Inline Comments Quality | 8% | WHY over WHAT, no stale TODOs |
| C10 | Cross-references & Linking | 5% | Connected docs are navigable docs |
| C11 | Testing & CI | 8% | Docs-as-code: lint, link-check, doc-tests |
| C12 | Maintenance Process | 5% | Ownership, review workflow, update cadence |

## DQI Thresholds

```
8.0-10.0 → Excellent  — exemplary documentation, industry reference
6.0-7.9  → Good       — solid with minor gaps, usable by newcomers
4.0-5.9  → Acceptable — functional but significant improvements needed
2.0-3.9  → Concerning — major gaps, docs actively misleading in places
0.0-1.9  → Critical   — documentation emergency
```

## Derived Metrics

**Doc Debt Score:**
```
Doc Debt = (Public Items − Documented Items) / Public Items × 100%

Thresholds:
  0-10%  → Green  (healthy)
  10-25% → Yellow (needs attention)
  25-50% → Orange (significant debt)
  50%+   → Red    (documentation emergency)
```

**Doc Maturity Level (inspired by "Docs for Developers"):**

| Level | Name | Criteria |
|-------|------|----------|
| 1 | Ad hoc | No process, docs written when someone complains |
| 2 | Reactive | Docs after features ship, often stale |
| 3 | Proactive | Docs part of DoD, reviewed in PRs |
| 4 | Managed | Metrics tracked, feedback loops, quality measured |
| 5 | Optimized | Continuous improvement, doc-tests, user research |

## Anti-Patterns

| Anti-Pattern | Category | Detection |
|-------------|----------|-----------|
| **Wall of Text** | C4 | Paragraphs > 5 sentences, no structure |
| **The Lie** | C3 | Examples that don't actually work |
| **Jargon Soup** | C4 | Undefined acronyms and domain terms |
| **Treasure Hunt** | C5 | Critical info buried deep, no navigation |
| **Time Capsule** | C3 | Docs last updated years ago, code changed since |
| **Copy-Paste Graveyard** | C3 | Duplicated content that drifts out of sync |
| **The Assumption** | C4 | Skipping prerequisites ("obviously you've already...") |
| **Condescension** | C4 | "Simply", "just", "easy", "obviously" |
| **Happy Path Only** | C6 | No error handling, no troubleshooting |
| **Orphan Pages** | C5 | Docs not linked from anywhere |
| **Version Amnesia** | C3 | No indication of which version the doc applies to |
| **Screenshot Dependency** | C3 | UI docs relying on screenshots that rot |

## Sources

### Books
- Bhatti, Corleissen et al. — *Docs for Developers*
- Cyrille Martraire — *Living Documentation*
- Daniele Procida — Diataxis framework (diataxis.fr)

### Standards & Guides
- Google Developer Documentation Style Guide
- Microsoft Writing Style Guide
- RFC 1574 (Rust doc conventions)
- Write the Docs community resources

### Tools
- Vale — prose quality linting (Google, Microsoft, write-good packages)
- lychee — fast link checking (Rust)
- markdownlint — markdown structure linting
- Spectral — OpenAPI doc linting
- cspell — spell checking

### YouTube / Podcasts
- Write the Docs Podcast — doc tooling, strategy, metrics
- I'd Rather Be Writing (Tom Johnson) — API doc quality, 75-point checklist
- Knowledgebase Ninja — developer relations and docs
- Daniele Procida — "What Nobody Tells You About Documentation" (Write the Docs talk)
