# CONTRACTS.md Template

> **When to read:** During Bootstrap Mode (Step 3) to generate CONTRACTS.md for a project that doesn't have one yet.

---

## File header

```markdown
# CONTRACTS.md — Functional Intentions & Behavioral Contracts

> This file records the **intended behavior** of critical functions and APIs.
> It is the reference against which code is compared to detect **silent semantic drift**:
> changes that compile, pass tests, but violate the original intention.
>
> **Maintained by:** [team/person]
> **Read by:** Claude Code (autophagy scan), code reviewers, QA
> **Update policy:** Update this file when a drift is intentional (new requirement).
> Never update it to match a bug — fix the code instead.
```

## Contract block template

```markdown
## {function_or_endpoint_name}({params})

**Module:** `{file_path}`
**Intention:** {1-3 sentences describing what this function SHOULD do, in natural language}

**Invariants:**
- {INV-1}: {formal property that must always hold}
- {INV-2}: {another property}
- {INV-N}: ...

**Preconditions:**
- {what must be true BEFORE calling this function}

**Postconditions:**
- {what must be true AFTER calling this function}

**Boundary behavior:**
- {what happens at empty input, max size, zero, null, etc.}

**Known drifts:**
- {YYYY-MM-DD}: {description of past drift} — {status: fixed/wontfix/investigating}

**Red flags to detect:**
- {specific code pattern that would indicate drift}
- {another pattern}
```

## Common invariant templates

Use these as starting points when writing invariants:

### Uniformity
```
{function} must behave identically for all values of {parameter} —
the only difference is which {entity} is targeted.
Any `if {parameter} == "{specific_value}"` branch is a red flag.
```

### Idempotency
```
{function}({function}(x)) == {function}(x) for all valid x.
Applying the function twice must produce the same result as once.
```

### Roundtrip / Reversibility
```
{reverse_function}({function}(x)) == x for all valid x.
Encoding then decoding must recover the original.
```

### Monotonicity
```
If x > y, then {function}(x) >= {function}(y).
The function preserves ordering.
```

### Conservation
```
{measure}(input) == {measure}(output) for all valid inputs.
The function preserves {quantity} (count, sum, set membership, etc.).
```

### Subset / Non-amplification
```
output ⊆ input (in terms of {dimension}).
The function never adds {information/data/permissions} beyond what was in the input.
```

### Commutativity
```
{function}(a, b) == {function}(b, a) for all valid a, b.
Parameter order does not affect the result.
```

### Determinism
```
{function}(x) always returns the same result for the same x.
No hidden state, no time dependency, no random component.
```

## Contract sizing guide

Not every function needs a contract. Focus on:

| Priority | Criteria | Examples |
|----------|---------|---------|
| **P0 — Must have** | Public API endpoints, core business logic, security-sensitive functions | `authenticate()`, `calculate_price()`, `mask()`, `transfer()` |
| **P1 — Should have** | Functions with non-obvious invariants, state machines, validators | `validate_input()`, `transition_state()`, `serialize()` |
| **P2 — Nice to have** | Internal utilities with tricky edge cases | `parse_date()`, `normalize()`, `deduplicate()` |
| **Skip** | Pure CRUD, simple getters/setters, framework glue | `get_user()`, `save()`, `to_string()` |

## Example: complete contract

```markdown
## mask(data, fields)

**Module:** `src/core/mask.rs`
**Intention:** Return `data` with every field listed in `fields` replaced by `***`.
Behavior must be identical regardless of which field is targeted — only the field
name determines what gets masked, never the field's content.

**Invariants:**
- INV-1 (Uniformity): mask(data, [a]) and mask(data, [b]) apply the same
  transformation — only the targeted field differs
- INV-2 (Idempotency): mask(mask(data, f), f) == mask(data, f)
- INV-3 (Non-mutation): fields NOT in `fields` must be byte-identical in output
- INV-4 (Content-independence): mask does not inspect or depend on the value
  of the field being masked, only its presence in `fields`

**Preconditions:**
- `data` is valid structured data (JSON, struct, map)
- `fields` is a non-empty list of field names

**Postconditions:**
- Every field in `fields` that exists in `data` is replaced by `***`
- Every field NOT in `fields` is unchanged
- Output structure is identical to input structure

**Boundary behavior:**
- Empty `fields` list: return `data` unchanged
- Field not in `data`: silently ignore (no error)
- Nested fields: only top-level matching (unless dotted notation `a.b.c`)

**Known drifts:**
- 2024-03-15: mask(data, [email]) did not mask sub-domain part — FIXED
- 2026-03-20: mask(data, [b]) applied an extra .trim() on the value before
  masking — INVESTIGATING (violates INV-1 and INV-4)

**Red flags to detect:**
- Any `if field == "b"` or `match field { "b" => ... }` inside mask()
- Any `.trim()`, `.lowercase()`, or transformation applied before/after masking
- Any branch that depends on the field's VALUE rather than its NAME
```
