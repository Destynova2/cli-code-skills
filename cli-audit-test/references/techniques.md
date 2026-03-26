# Technique Detection Heuristics

> **When to read:** During Step 2 (classify) and Step 3 (score D2) to identify which test design techniques are used in the codebase.

---

## Detection by Technique

### Equivalence Partitioning (EP)

**What it is:** Dividing inputs into classes where all values in a class should behave the same.

**Detection signals:**
- Test names containing "valid", "invalid", "class", "category", "group"
- Multiple test cases with representative values from different input ranges
- Comments mentioning "partition", "equivalence", "class"
- Test data organized by input categories

**Example evidence:**
```
tests/auth/valid-credentials.hurl
tests/auth/invalid-credentials.hurl
tests/auth/expired-token.hurl
```

---

### Boundary Value Analysis (BVA)

**What it is:** Testing at the edges of equivalence partitions (min, min+1, max-1, max, just-outside).

**Detection signals:**
- Test names with "limit", "boundary", "max", "min", "edge", "threshold"
- Numeric values at boundaries: 0, 1, -1, MAX_INT, empty string, max length
- Percentage tests: 0%, 1%, 99%, 100%, 101%
- Size/count tests: 0 items, 1 item, max items, max+1 items

**Example evidence:**
```
test_budget_at_79_percent    → below threshold
test_budget_at_80_percent    → at threshold
test_budget_at_100_percent   → at limit
test_budget_at_101_percent   → above limit
```

---

### Decision Table

**What it is:** Testing combinations of conditions and their expected outcomes (truth table).

**Detection signals:**
- Files named `*decision*`, `*policy*`, `*rule*`, `*matrix*`
- Truth tables in comments or documentation
- Tests with combinations of boolean conditions
- Policy-based test organization

**Example evidence:**
```
tests/policies/
  admin-active-mfa.hurl        → admin=true, active=true, mfa=true
  admin-active-no-mfa.hurl     → admin=true, active=true, mfa=false
  user-inactive.hurl           → admin=false, active=false
```

---

### State Transition

**What it is:** Testing transitions between states (valid transitions, invalid transitions, sequences).

**Detection signals:**
- Files named `*state*`, `*transition*`, `*lifecycle*`, `*circuit*`, `*workflow*`
- State machine diagrams in docs
- Tests that follow sequences: created → active → suspended → deleted
- Circuit breaker tests: closed → open → half-open

**Example evidence:**
```
tests/circuit-breaker/
  closed-to-open.hurl
  open-to-half-open.hurl
  half-open-to-closed.hurl
  invalid-transition.hurl
```

---

### Pairwise / Combinatorial

**What it is:** Testing parameter interactions using combinatorial reduction (instead of exhaustive N×M).

**Detection signals:**
- PICT files (`*.pict`, `pict/` directory)
- Pairwise test generation tool in dependencies
- Comments mentioning "pairwise", "combinatorial", "all-pairs"
- Test matrices with specific combination patterns

**Example evidence:**
```
pict/feature-interactions.pict
tests/generated/pairwise-combinations.csv
```

---

### Use Case / Scenario Testing

**What it is:** End-to-end scenarios that represent real user workflows.

**Detection signals:**
- Files named `*scenario*`, `*flow*`, `*happy*`, `*journey*`, `*workflow*`
- Multi-step test files (e.g., Hurl files with multiple requests in sequence)
- BDD-style tests (Given/When/Then)
- User story references in test names

**Example evidence:**
```
tests/e2e/
  user-registration-flow.hurl
  checkout-happy-path.hurl
  admin-dashboard-scenario.hurl
```

---

### Error Guessing / Fuzz Testing

**What it is:** Experience-based testing and random input generation to find unexpected failures.

**Detection signals:**
- Files named `*fuzz*`, `*edge*`, `*chaos*`, `*monkey*`
- Fuzz targets in `fuzz/` directory
- Property-based testing frameworks (proptest, hypothesis, fast-check)
- Random/generated test data

**Example evidence:**
```
fuzz/
  fuzz_parser.rs
  fuzz_api_input.rs
tests/edge-cases/
  unicode-input.hurl
  empty-body.hurl
  oversized-payload.hurl
```

---

### Exploratory Testing

**What it is:** Structured but unscripted testing guided by charters and time-boxes.

**Detection signals:**
- Files named `*exploratory*`, `*charter*`, `*session*`
- SBTM (Session-Based Test Management) references
- Exploration notes or debriefs
- Time-boxed testing mentioned in test plan

**Example evidence:**
```
docs/test-charters/
  charter-001-new-auth-flow.md
  session-report-2024-01-15.md
```

---

## Quick Reference Table

| Technique | Key file patterns | Key test name patterns |
|-----------|------------------|----------------------|
| EP | - | `*valid*`, `*invalid*`, `*class*` |
| BVA | - | `*limit*`, `*boundary*`, `*max*`, `*min*`, `*edge*` |
| Decision Table | `*decision*`, `*policy*` | `*rule*`, `*combination*` |
| State Transition | `*state*`, `*circuit*` | `*transition*`, `*lifecycle*` |
| Pairwise | `*.pict`, `pict/` | `*pairwise*`, `*combination*` |
| Scenario | `*scenario*`, `*flow*` | `*happy*`, `*journey*`, `*e2e*` |
| Fuzz | `fuzz/`, `*fuzz*` | `*edge*`, `*chaos*`, `*random*` |
| Exploratory | `*charter*`, `*session*` | `*exploratory*` |
