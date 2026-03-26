# Code Quality Index (CQI) — Scoring Framework

> **When to read:** Step 4 (Compute final score).

---

## CQI Formula

```
CQI = Σ(wᵢ × sᵢ) / Σ(wᵢ) × 10

Where:
  wᵢ = weight of category i
  sᵢ = score of category i (0.0 to 1.0)
```

## Scoring Guide

| Score | Meaning |
|-------|---------|
| 0.0 | Absent — no evidence of this practice |
| 0.25 | Initial — sporadic, inconsistent |
| 0.5 | Basic — present but incomplete, significant gaps |
| 0.75 | Good — consistent, minor issues remain |
| 1.0 | Excellent — exemplary, could serve as reference |

## Category Weights

| # | Category | Weight | Rationale |
|---|----------|--------|-----------|
| C1 | Naming & Readability | 8% | Foundation of comprehension |
| C2 | Functions & Cognitive Complexity | 12% | #1 predictor of bugs (SonarSource research) |
| C3 | Module Design (deep vs shallow) | 10% | Determines long-term evolvability (Ousterhout) |
| C4 | DRY & Change Amplification | 8% | #1 source of inconsistency bugs |
| C5 | Error Handling & Robustness | 10% | Production fails on error paths (SonarQube Reliability) |
| C6 | Type Safety & Language Idioms | 8% | Leveraging type system prevents bug classes |
| C7 | Comments & Public API Docs | 5% | Good code is self-documenting; public API still matters |
| C8 | Test Quality | 12% | Safety net for all other categories |
| C9 | Security & Input Validation | 10% | SonarQube Security axis; parse-don't-validate |
| C10 | Immutability & State Management | 7% | Mutable shared state = concurrency bugs |
| C11 | Cognitive Load & Control Flow | 5% | Deep nesting, boolean complexity |
| C12 | Dependencies & Architecture | 5% | Coupling, circular deps (Tornhill) |

## CQI Thresholds

```
8.0-10.0 → Excellent  — production-ready exemplary code
6.0-7.9  → Good       — solid with minor issues, ship-ready
4.0-5.9  → Acceptable — needs improvement, technical debt accumulating
2.0-3.9  → Concerning — significant quality issues, refactoring needed
0.0-1.9  → Critical   — major rewrite or restructuring required
```

## Derived Metrics

**Tech Debt Estimate (SQALE-inspired):**
```
Debt = Σ (critical_findings × 2h + flag_findings × 0.5h)
Debt Ratio = Debt / (total_lines × 0.5h_per_line) × 100%

Thresholds:
  A = 0-5%   (healthy)
  B = 6-10%  (minor debt)
  C = 11-20% (significant)
  D = 21-50% (alarming)
  E = 50%+   (critical)
```

**Severity Classification (SonarQube-aligned):**

| Severity | Definition | Example |
|----------|-----------|---------|
| Blocker | Will crash or corrupt data | `unwrap()` on user input |
| Critical | Security vulnerability or major logic error | SQL injection, hardcoded secret |
| Major | Significant maintainability issue | 80-line function, God module |
| Minor | Style or minor quality issue | Missing doc comment, verbose name |
| Info | Suggestion, not a violation | "Consider using an iterator here" |

## Anti-Patterns (Named, from Fowler/Mantyla taxonomy)

| Anti-Pattern | Category | Detection |
|-------------|----------|-----------|
| **Feature Envy** | C12 | Method uses another class's data more than its own |
| **Shotgun Surgery** | C4 | One logical change touches 5+ files |
| **God Class / Blob** | C3 | File > 1000 lines, module does too many things |
| **Primitive Obsession** | C6 | Using strings/ints where domain types should exist |
| **Long Parameter List** | C2 | Function with > 5 parameters |
| **Data Clumps** | C4 | Same group of 3+ parameters appears in multiple functions |
| **Dead Code / Lava Flow** | C7 | Commented-out code, unreachable branches |
| **Inappropriate Intimacy** | C12 | Two modules know too much about each other's internals |
| **Speculative Generality** | C4 | Abstraction built for future use that never came |
| **Middle Man** | C3 | Class that delegates everything, adds nothing (shallow module) |

## Comparative Benchmarks

| Project Type | Typical CQI | Target |
|-------------|-------------|--------|
| Startup MVP / prototype | 3.0-5.0 | 5.0+ |
| Internal tool | 4.0-6.0 | 6.0+ |
| Production SaaS | 5.0-7.0 | 7.0+ |
| Open-source library | 6.0-8.0 | 8.0+ |
| Financial / Safety-critical | 7.0-9.0 | 9.0+ |

## Sources

### Books
- Robert C. Martin — *Clean Code*
- John Ousterhout — *A Philosophy of Software Design*
- Martin Fowler — *Refactoring* (2nd ed)
- Adam Tornhill — *Software Design X-Rays*, *Your Code as a Crime Scene*
- Michael Feathers — *Working Effectively with Legacy Code*

### Tools & Models
- SonarQube Quality Model (Reliability, Security, Maintainability)
- SonarSource Cognitive Complexity whitepaper
- SQALE Method for Technical Debt
- CodeScene Code Health model
- Clippy lint categories (Rust)

### YouTube / Podcasts
- CodeAesthetic — software design principles, animated deep dives
- ThePrimeagen — performance, Rust, code quality opinions
- ArjanCodes — design patterns, code roasts
- Software Engineering Radio — professional SE topics
- Tech Lead Journal — ep. 241: Tornhill on code health
