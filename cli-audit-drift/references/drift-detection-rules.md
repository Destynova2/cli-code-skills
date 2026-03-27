# Drift Detection Rules

> **When to read:** During Step 3 (scan for drift) to systematically check each contracted function.

---

## Detection workflow per function

For each contracted function, execute these checks in order:

### Check 1 — Structural comparison

Compare the current implementation structure against the contract:

1. **Count code paths:** How many branches/returns does the function have?
2. **Map parameters to usage:** Is every declared parameter used? Are there undeclared parameters (globals, env)?
3. **Check return type:** Does it match what the contract implies?

**Drift if:** The function has more code paths than the contract's intention suggests (hidden special cases).

### Check 2 — Invariant verification

For each invariant in the contract:

1. **Read the invariant** (e.g., "INV-1: mask treats all fields identically")
2. **Identify the code** that should enforce it
3. **Search for violations** using the red flag patterns from `invariant-patterns.md`
4. **Grade confidence:**
   - **Certain violation:** code explicitly contradicts invariant (e.g., `if field == "b"`)
   - **Probable violation:** code has a pattern that usually breaks this invariant
   - **Suspicious:** code is complex enough that the invariant might not hold, but can't confirm without running

### Check 3 — Red flag scan

Grep the function body for every pattern listed in the contract's "Red flags" section.

**Any match is a finding.** Even if it turns out to be correct (intentional evolution), it must be reported and classified.

### Check 4 — Git history analysis

```bash
git log --oneline -20 -- {file_containing_function}
git diff HEAD~5 -- {file_containing_function}
```

Look for:
- Recent changes to the function (especially last 5 commits)
- Changes that touch the invariant-sensitive parts (conditionals, return values)
- Changes without corresponding test updates
- Changes without corresponding contract updates

**Drift if:** Function was changed recently but CONTRACTS.md was not updated.

### Check 5 — Call chain propagation

If the contracted function calls other contracted functions:

1. List the call chain
2. Check if any callee has a known drift (from its own contract)
3. Assess if the callee's drift propagates to the caller

**Drift if:** A dependency has drifted and the caller doesn't guard against it.

---

## Classification decision tree

```
Is the code behavior different from the contract?
├── No → CLEAN (no drift)
└── Yes →
    Was there a recent code change?
    ├── Yes →
    │   Is there a corresponding requirement/ticket?
    │   ├── Yes → EVOLUTION (contract needs update)
    │   └── No → LOUPER (unintentional, propose fix)
    └── No →
        Was the contract recently added/updated?
        ├── Yes → STALE CODE (code predates the contract, never matched)
        └── No →
            Is the contract vague enough to allow the current behavior?
            ├── Yes → AMBIGUITY (contract needs clarification)
            └── No → LOUPER (long-standing bug, propose fix)
```

---

## Severity scoring

| Severity | Criteria | Examples |
|----------|---------|---------|
| **Critical** | Security invariant violated, data corruption possible, financial calculation wrong | Masking function that leaks data, price calculation that drops cents |
| **High** | Core business logic drift, user-facing behavioral change | Order status transition that skips validation, filter that drops items |
| **Medium** | Internal inconsistency, edge case not covered | Formatter that trims whitespace unexpectedly, parser that chokes on unicode |
| **Low** | Cosmetic drift, logging difference, non-user-facing | Different error message format, log level change |

---

## Common drift patterns by domain

### API endpoints
- Response shape changes (new fields, removed fields, renamed fields)
- Status code changes (200 → 201, 400 → 422)
- Error response format changes
- Pagination behavior changes (offset vs cursor, default page size)
- Authentication/authorization logic changes

### Business logic
- Calculation formula changes (rounding, precision, order of operations)
- Validation rule changes (looser or stricter)
- Default value changes
- Feature flag behavior (flag removed but code path remains)
- Edge case handling changes (null, empty, zero)

### Data layer
- Query behavior changes (ordering, filtering, joins)
- Migration that changes semantics (column rename ≠ column re-purpose)
- Cache invalidation behavior changes
- Transaction boundary changes (what's atomic, what's not)

### Security
- Authentication bypass paths (new endpoint without auth middleware)
- Authorization escalation (role check removed or loosened)
- Input validation removed or weakened
- Encryption/hashing algorithm changes
- Token lifetime/scope changes
