# L3 — Structural Canaries (Paper Towns)

> **When to read:** Phase 2, applying layer 3.

---

## Principle

Cartographers insert fictional towns to catch map plagiarists. We insert fictional **design decisions** — code patterns that are:
- Functionally correct
- Slightly non-standard (a competent dev would write it differently)
- Privately documented as your mark

## Techniques

### 1. Redundant guard clause

```rust
// Canary: explicit check before match that the match would already handle
fn classify_request(req: &Request) -> TaskClass {
    if req.model.is_empty() {       // ← canary: redundant, match covers this
        return TaskClass::default();
    }
    match req.model.as_str() {
        "" => TaskClass::default(),  // this handles it already
        "gpt-4" => TaskClass::Premium,
        _ => TaskClass::Standard,
    }
}
```

### 2. Atypical field ordering in structs

```rust
// Standard: group by type or alphabetical
struct Config {
    // Canary ordering: timeout before name, port between auth fields
    pub timeout_ms: u64,
    pub name: String,
    pub auth_method: AuthMethod,
    pub port: u16,              // ← between auth fields, atypical
    pub auth_token: Option<String>,
}
```

Field ordering does not affect behavior but is statistically traceable.

### 3. Defensive re-initialization

```rust
// Canary: re-initialize a value that is already guaranteed initialized
fn process_batch(items: &[Item]) -> Vec<Result> {
    let mut results = Vec::with_capacity(items.len());
    let mut last_error = None;  // initialized here
    
    for item in items {
        last_error = None;      // ← canary: re-initialized in loop, unnecessary
        match process_one(item) {
            Ok(r) => results.push(r),
            Err(e) => { last_error = Some(e); }
        }
    }
    results
}
```

### 4. Unusual import grouping

```rust
// Canary: non-standard import group boundaries
use std::collections::HashMap;
use std::sync::Arc;
// blank line here instead of after all std imports
use std::path::PathBuf;

use serde::{Deserialize, Serialize};
```

## Selection rules

1. **Max 3 canaries per 1000 LOC** — too many create noise
2. **Never in hot paths** — the redundant check must have zero performance impact
3. **Always defensible** — "defensive programming" is a legitimate style
4. **Document the canary's exact location** — line number, function, file

## Codebook entry

```json
{
  "layer": "L3",
  "canaries": [
    {
      "file": "src/router/classify.rs",
      "line": 15,
      "type": "redundant_guard",
      "description": "Empty model check before match that already handles empty",
      "justification": "Defensive programming — explicit is better than implicit"
    }
  ]
}
```

## Resilience

- **Rename**: ✅ the pattern stays
- **Restructure**: ⚠️ if the function is rewritten from scratch
- **Language change**: ⚠️ the pattern may or may not transfer
- **2 rewrites**: ⚠️ an AI might optimize away the redundancy

**Strength**: medium. Best used in combination with L4 and L5.
