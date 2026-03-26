# Quality Scoring — Design Completeness Index (DCI-LLD)

> **When to read:** Load this file after writing the LLD to score its completeness. Contains the DCI-LLD formula, 12 checklist items with weights, thresholds, and derived metrics.

---

After writing, score the LLD using the DCI-LLD formula (inspired by cli-forge-doc's DCI).

## DCI-LLD Formula

```
DCI = Σ(wᵢ × sᵢ) / Σ(wᵢ) × 10

Where:
  wᵢ = weight of item i (importance, adjusted by tier)
  sᵢ = score of item i (0.0 to 1.0)
```

## Scoring Guide

| Score | Meaning |
|-------|---------|
| 0.0 | Absent — not addressed |
| 0.25 | Stub — mentioned but a dev can't implement from it |
| 0.5 | Present but incomplete — missing edge cases or error paths |
| 0.75 | Good — a dev could implement, some questions remain |
| 1.0 | Excellent — unambiguous, a dev can implement without asking questions |

## Checklist Items (with weights)

| # | Item | Weight | What to check |
|---|------|--------|---------------|
| 1 | Component architecture (C4 L3) | 5 | Layers, responsibilities, dependency direction clear |
| 2 | API contracts | 5 | All endpoints, request/response schemas, error codes, pagination |
| 3 | Database schema | 5 | ER diagram, column types, indexes with rationale |
| 4 | Sequence diagrams | 4 | Happy path + key error paths covered |
| 5 | Class/module design | 4 | Interfaces defined, SOLID visible, patterns justified |
| 6 | Error handling | 4 | Taxonomy, retry/circuit breaker config, error format |
| 7 | State machines | 3 | Transition table with guards and side effects |
| 8 | Security (STRIDE) | 3 | Component-level threats identified, input validation rules |
| 9 | Testability | 3 | DI points, mock boundaries, test strategy per level |
| 10 | Domain model | 3 | Entities, VOs, aggregates, invariants |
| 11 | Concurrency | 2 | Shared state, synchronization mechanisms, race mitigations |
| 12 | Migration plan | 2 | Phases, rollback strategy (if brownfield) |

**Tier adjustment:** Items not included per the mitosis matrix get weight = 0.

## DCI-LLD Thresholds

```
8.0-10.0 → Excellent — a dev can implement without asking questions
6.0-7.9  → Good — implementable with minor clarifications needed
4.0-5.9  → Acceptable — major areas need more detail
2.0-3.9  → Incomplete — too many unknowns to start coding
0.0-1.9  → Stub — not a usable design document
```

## Derived Metrics

**Contract Coverage:**
```
CC = documented API endpoints / total endpoints in code × 100%
Target: 100% for public API, ≥ 80% for internal
```

**Testability Score:**
```
TS = DI points documented / total external dependencies × 100%
Target: ≥ 90% (every dependency should be injectable)
```

**Implementation Ambiguity:**
```
IA = questions a dev would need to ask / total design decisions × 100%
Target: < 10% (the doc should answer almost everything)
```

Output the DCI-LLD score table at the end of the document.
