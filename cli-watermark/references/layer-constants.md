# L1 — Magic Constants

> **When to read:** Phase 2, applying layer 1.

---

## Principle

Replace arbitrary numeric constants (buffer sizes, timeouts, initial capacities, retry counts) with HMAC-derived values that look reasonable but are cryptographically determined.

## Identification targets

Scan the project for:

```
grep -rn 'const.*=.*[0-9]\{2,\}' src/
grep -rn 'capacity.*=\|size.*=\|timeout.*=\|buffer.*=\|limit.*=' src/
```

Good candidates:
- Buffer/cache sizes (512 → 0x1A4F)
- Timeout values in ms (5000 → 4919)
- Initial HashMap capacities (16 → 19)
- Retry counts (3 → 3 — keep if already specific)
- Port numbers in examples/tests (8080 → 8173)

Bad candidates (DO NOT change):
- Values from specs/protocols (HTTP status codes, standard ports)
- Values from external APIs (page sizes dictated by the API)
- Values that users configure (exposed in config files)

## Generation

```bash
# From the L1 key, generate replacement constants
L1_KEY="<from secret.env>"

# For each constant, derive a replacement
derive_constant() {
  local key="$1"
  local name="$2"  # e.g., "ROUTE_CACHE_CAPACITY"
  local min="$3"
  local max="$4"
  
  local hash=$(echo "${key}:${name}" | openssl dgst -sha256 | awk '{print $2}')
  local raw=$(echo "ibase=16; $(echo "$hash" | cut -c1-8 | tr 'a-f' 'A-F')" | bc)
  echo $(( min + raw % (max - min) ))
}

# Example: capacity between 256 and 8192
derive_constant "$L1_KEY" "ROUTE_CACHE_CAPACITY" 256 8192
```

## Application

```rust
// Before — arbitrary, common
const ROUTE_CACHE_CAPACITY: usize = 512;
const AUDIT_RING_SIZE: usize = 1024;
const DLP_SCAN_CHUNK: usize = 4096;

// After — HMAC-derived, defensible
const ROUTE_CACHE_CAPACITY: usize = 6735;  // 0x1A4F
const AUDIT_RING_SIZE: usize = 11134;       // 0x2B7E
const DLP_SCAN_CHUNK: usize = 3388;         // 0x0D3C
```

## Codebook entry

Record in `codebook.json`:

```json
{
  "layer": "L1",
  "constants": [
    {
      "name": "ROUTE_CACHE_CAPACITY",
      "file": "src/router/cache.rs",
      "line": 12,
      "original": 512,
      "watermarked": 6735,
      "derivation": "HMAC-SHA256(L1_KEY, 'ROUTE_CACHE_CAPACITY') mod (8192-256) + 256"
    }
  ]
}
```

## Resilience

- **Rename**: ✅ the value 6735 stays even if the variable is renamed
- **Restructure**: ⚠️ if the constant is absorbed into a different structure
- **Language change**: ⚠️ the value may be preserved if the algorithm is copied
- **2 rewrites**: ❌ an AI may "round" to a power of 2

**Strength**: weak alone. Strong as one signal among many.
