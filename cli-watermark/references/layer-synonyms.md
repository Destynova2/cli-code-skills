# L2 — Synonym Encoding

> **When to read:** Phase 2, applying layer 2. Weakest layer — use only as additional signal.

---

## Principle

In naming, `fetch`/`get`/`retrieve`/`load` are semantically equivalent. Choose systematically according to a bitstream derived from your secret, encoding your identity in the distribution of synonym choices across N functions.

## Codebook

```json
{
  "bit_pairs": [
    {"bit": 0, "options": ["fetch", "get"]},
    {"bit": 1, "options": ["emit", "send"]},
    {"bit": 2, "options": ["handle", "process"]},
    {"bit": 3, "options": ["build", "create"]},
    {"bit": 4, "options": ["remove", "delete"]},
    {"bit": 5, "options": ["check", "verify"]},
    {"bit": 6, "options": ["load", "read"]},
    {"bit": 7, "options": ["store", "save"]}
  ]
}
```

Signature binary: derive from L2_KEY, e.g., `0b10110101`:
- bit 0 = 1 → use "get" (not "fetch")
- bit 1 = 0 → use "emit" (not "send")
- bit 2 = 1 → use "process" (not "handle")
- etc.

## Application

Only apply to NEW code (not renaming existing functions — that would break the API).

Over N=20 functions, the probability of the exact same synonym distribution by chance is ~(1/2)^20 ≈ 1 in a million.

## Resilience

- **Rename**: ❌ — this IS names
- **Clean code**: ❌ — a linter may normalize
- **Language change**: ❌ — names don't transfer
- **Statistical detection**: ⚠️ — works over large N, unreliable per-function

**Use only as supplementary evidence, never as primary proof.**
