# Quality Scoring — Architecture Completeness Index (ACI)

> **When to read:** After writing the HLD, to score its completeness. Inspired by cli-forge-doc's DCI.

---

## ACI Formula

```
ACI = Σ(wᵢ × sᵢ) / Σ(wᵢ) × 10

Where:
  wᵢ = weight of item i (importance, adjusted by tier)
  sᵢ = score of item i (0.0 to 1.0)
```

## Scoring Guide

| Score | Meaning |
|-------|---------|
| 0.0 | Absent — not addressed |
| 0.25 | Stub — mentioned but no substance |
| 0.5 | Present but incomplete — missing key elements |
| 0.75 | Good — covers most aspects, minor gaps |
| 1.0 | Excellent — complete, justified, with tradeoffs |

## Checklist Items (with weights)

| # | Item | Weight | What to check |
|---|------|--------|---------------|
| 1 | Problem statement & goals | 5 | Clear problem, measurable goals, non-goals stated |
| 2 | System context diagram (C4 L1) | 5 | All actors and external systems, arrows labeled |
| 3 | Container architecture (C4 L2) | 5 | Components, protocols, responsibilities clear |
| 4 | Capacity estimation | 4 | Numbers with formulas, assumptions stated, sanity-checked |
| 5 | Technology justification | 4 | Every tech choice has a "why", not just listed |
| 6 | ADRs | 5 | 1+ ADR per major decision, alternatives considered |
| 7 | Security architecture | 4 | Auth, encryption, trust boundaries, compliance |
| 8 | Data architecture | 3 | Storage strategy, data flow, retention/backup |
| 9 | Deployment architecture | 3 | Infra, CI/CD, environments, scaling strategy |
| 10 | NFRs / SLOs | 4 | Measurable targets, not vague aspirations |
| 11 | Failure modes | 3 | Probability × impact, mitigations, detection |
| 12 | Tradeoff analysis | 3 | Explicit tradeoffs, not just "we chose X" |

**Tier adjustment:** For tier S/M, items with weight that don't apply per the mitosis matrix get weight = 0 (excluded from both numerator and denominator).

## ACI Thresholds

```
8.0-10.0 → Excellent — production-ready design doc
6.0-7.9  → Good — solid with minor gaps, iterate
4.0-5.9  → Acceptable — functional but needs work
2.0-3.9  → Incomplete — major sections missing or vague
0.0-1.9  → Stub — not a usable design document
```

## Derived Metrics

**Decision Coverage:**
```
DC = ADRs documented / major decisions identified × 100%
Target: ≥ 80%
```

**Estimation Confidence:**
```
EC = assumptions explicitly stated / total numbers used × 100%
Target: ≥ 90% (every number needs a source or assumption)
```

Output the ACI score table at the end of the document.
