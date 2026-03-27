# Invariant Patterns — Detection & Classification

> **When to read:** During Step 1 (parse contracts) and Step 3 (scan for drift) to identify which invariant categories apply and how to detect violations.

---

## Invariant taxonomy

### Category 1 — Symmetry invariants

Functions that must treat all values of a parameter uniformly.

**Pattern:** `f(x, a) ≈ f(x, b)` — same transformation, different target.

**Drift signals:**
- `if param == "specific_value"` — branch on parameter identity
- `match param { "a" => ..., "b" => different_thing }` — non-uniform dispatch
- Different code paths for values that should be interchangeable
- Hardcoded lists that exclude certain values

**Common in:** masking, filtering, permission checks, field selectors, formatters

**How to verify:**
1. Grep for conditionals on the parameter that should be uniform
2. Check that all code paths through the function are structurally identical
3. Look for `special_cases`, `exceptions`, `overrides` variables

---

### Category 2 — Idempotency invariants

Applying the function twice yields the same result as once.

**Pattern:** `f(f(x)) == f(x)`

**Drift signals:**
- Function accumulates state (counter++, append to list)
- Function has side effects that compound (double-write, double-log)
- Normalization that changes already-normalized input

**Common in:** normalizers, formatters, sanitizers, cache operations, idempotent API endpoints (PUT)

**How to verify:**
1. Check for mutable state that persists across calls
2. Check for append/push/increment operations
3. Trace what happens when output is fed back as input

---

### Category 3 — Roundtrip invariants

Encoding then decoding (or vice versa) recovers the original.

**Pattern:** `decode(encode(x)) == x`

**Drift signals:**
- Lossy transformation (truncation, rounding, lossy encoding)
- Encoding adds metadata that decoder doesn't strip
- Character encoding issues (UTF-8 → ASCII → UTF-8 loses characters)
- Serialization that drops optional/null fields

**Common in:** serializers/deserializers, encoders/decoders, import/export, API request/response mapping

**How to verify:**
1. Check for precision loss (float→int→float, datetime→string→datetime)
2. Check for null/undefined handling asymmetry
3. Check for field ordering assumptions

---

### Category 4 — Conservation invariants

A measurable quantity is preserved by the function.

**Pattern:** `measure(input) == measure(output)`

**Drift signals:**
- Items silently dropped or duplicated
- Totals that don't add up (financial: sum of line items ≠ total)
- Set operations that lose elements
- Pagination that skips or repeats items

**Common in:** financial calculations, set transformations, data pipelines, aggregations, splits/merges

**How to verify:**
1. Count elements before and after
2. Sum quantities before and after
3. Check for off-by-one in pagination/slicing
4. Verify that filter + complement = original set

---

### Category 5 — Monotonicity / ordering invariants

The function preserves or respects an ordering.

**Pattern:** `x > y → f(x) >= f(y)`

**Drift signals:**
- Sort that uses unstable comparison
- Priority queue that doesn't respect declared priority
- Version comparison that breaks on edge cases (1.9 vs 1.10)
- Ranking algorithm that doesn't preserve strict ordering

**Common in:** sorting, ranking, priority queues, comparators, version resolution

**How to verify:**
1. Test with boundary values where ordering flips
2. Check comparison operators for off-by-one
3. Verify transitivity: if a > b and b > c, then a > c

---

### Category 6 — Non-amplification / subset invariants

Output never contains more than what was in the input.

**Pattern:** `output ⊆ input` (along some dimension)

**Drift signals:**
- Function adds default values not present in input
- Logging/error messages that leak internal data
- Masking function that reveals information through error messages
- Filter that adds unmatched items as "other"

**Common in:** masking, filtering, access control, data projection, redaction

**How to verify:**
1. Check every field in output — was it in input?
2. Check error paths — do they leak information?
3. Verify that "not found" and "not authorized" are indistinguishable (for security)

---

### Category 7 — Determinism invariants

Same input always produces same output.

**Pattern:** `f(x) == f(x)` across time and context

**Drift signals:**
- Hidden dependency on current time (`now()`, `Date.now()`)
- Random components without seed (`rand()`, `uuid()`)
- Dependency on global/shared mutable state
- Dependency on environment variables or config that can change
- Database-dependent ordering (no explicit ORDER BY)

**Common in:** pure functions that should be pure, cache key generation, hash functions, test fixtures

**How to verify:**
1. Grep for `now()`, `random()`, `uuid()`, `env::var()`
2. Check for mutable static/global state
3. Verify database queries have explicit ordering

---

### Category 8 — Temporal / state machine invariants

State transitions follow declared rules.

**Pattern:** `state(t+1) ∈ allowed_transitions(state(t))`

**Drift signals:**
- Direct state assignment bypassing the transition function
- Missing guard clauses on transitions
- State that can be set to any value via API
- Parallel mutations that break transition atomicity

**Common in:** workflows, circuit breakers, order processing, user account states, payment processing

**How to verify:**
1. Find all places where state is mutated
2. Check that each mutation goes through the transition function
3. Verify that impossible transitions are rejected (not just unimplemented)
4. Check for race conditions in concurrent transitions

---

## Red flag cheat sheet

Quick patterns to grep for when scanning contracted functions:

| Red flag pattern | Suggests violation of | Example |
|-----------------|----------------------|---------|
| `if field == "specific"` | Symmetry (Cat. 1) | `if field == "b" { value.trim() }` |
| `counter += 1` in pure function | Idempotency (Cat. 2) | `call_count += 1; return result` |
| `.truncate()`, `.round()` in codec | Roundtrip (Cat. 3) | `encode(x).truncate(255)` |
| `items.len()` differs in/out | Conservation (Cat. 4) | `filter()` that drops without counting |
| `sort()` without comparator | Monotonicity (Cat. 5) | `items.sort()` on non-Ord type |
| Output has fields not in input | Non-amplification (Cat. 6) | `response.internal_id = ...` |
| `now()`, `random()` | Determinism (Cat. 7) | `cache_key = format!("{}-{}", id, now())` |
| Direct field assignment `.state =` | State machine (Cat. 8) | `order.state = "shipped"` bypassing FSM |
