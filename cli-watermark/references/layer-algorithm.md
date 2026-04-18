# L4 — Algorithmic Fingerprint

> **When to read:** Phase 2, applying layer 4. This is the strongest code-level layer.

---

## Principle

Choose a mathematically defensible but statistically rare algorithmic decision. An AI rewrite preserves the algorithm — it reformulates, it doesn't reinvent.

## Techniques

### 1. Rare prime in formula

Use a number that is:
- Mathematically notable (so it's defensible, not suspicious)
- Rare in the domain (so coincidence is improbable)

```rust
// 65537 = Fermat prime F4 (2^16 + 1)
// Defensible: "good for modular arithmetic properties"
// Rare: nobody uses F4 for score normalization
score = (latency / 65537.0) * w1 + (cost / 1337.0) * w2
```

Other good candidates:
- `1597` — Fibonacci prime
- `6737` — safe prime (3368×2 + 1)
- `0.61803` — golden ratio (φ-1), memorable and defensible
- `2.71828` — Euler's number, defensible for exponential decay

### 2. Non-standard wrap pattern

```rust
// Standard ring buffer:
let idx = self.head % self.capacity;

// Watermarked — XOR rotation before modulo
// Functionally identical for power-of-two capacities
// Structurally distinctive
let idx = (self.head ^ (self.head >> 7)) % self.capacity;
```

The XOR rotation travels with the algorithm through any rewrite.

### 3. Deliberate order of operations

```rust
// Standard: check A then B
if a && b { ... }

// Watermarked: check B then A (same result for pure conditions)
// Documented privately as your mark
if b && a { ... }
```

Short-circuit evaluation may differ for impure conditions — only use on pure predicates.

### 4. Non-standard normalization pivot

```rust
// Standard: normalize to [0, 1]
let norm = value / max_value;

// Watermarked: normalize with a specific pivot point
let norm = (value - PIVOT) / (max_value - PIVOT);
// where PIVOT = 0.23606... (√(1/18), defensible as "noise floor")
```

## Selection criteria

A good algorithmic fingerprint is:
1. **Correct** — produces the same or equivalent results
2. **Defensible** — has a plausible engineering justification
3. **Rare** — no other dev would independently make this exact choice
4. **Stable** — survives refactoring and language change
5. **Documentable** — you can explain to a judge why it's not accidental

## Codebook entry

```json
{
  "layer": "L4",
  "fingerprints": [
    {
      "location": "src/router/scoring.rs:42",
      "technique": "rare_prime_normalization",
      "value": 65537,
      "justification": "Fermat prime F4, modular arithmetic properties",
      "probability_of_coincidence": "< 1/10000 (estimated)",
      "survives_rewrite": true,
      "survives_lang_change": true
    }
  ]
}
```

## Resilience

- **Rename**: ✅ numbers don't have names
- **Restructure**: ✅ the formula stays
- **Language change**: ✅ the algorithm is copied
- **2 rewrites + clean code**: ✅ an AI won't replace 65537 with something else — it's already clean
- **3+ rewrites**: ✅ mathematically invariant

**This is the layer that survives everything except a complete algorithm rewrite.**
