# L7 — Published Test Vectors

> **When to read:** Phase 2, applying layer 7.

---

## Principle

Publish precise input/output pairs for your system BEFORE your competitor can copy them. If their implementation produces the same output on your inputs, their computation path is yours.

## Construction

### 1. Choose canonical inputs

Pick inputs that exercise your core algorithm — not trivial edge cases.

```json
[
  {
    "name": "canonical_routing",
    "input": {"model": "gpt-4", "task": "classify", "seed": 42},
    "expected_output": {"route": "fast", "score": 0.73418}
  },
  {
    "name": "edge_fallback",
    "input": {"model": "unknown-model", "task": "generate", "seed": 7},
    "expected_output": {"route": "default", "score": 0.50000}
  },
  {
    "name": "multi_route",
    "input": {"model": "claude-3", "task": "analyze", "seed": 1337},
    "expected_output": {"route": "balanced", "score": 0.61803}
  }
]
```

### 2. Precision matters

- Use 5 decimal places (not 2, not 8)
- The exact precision encodes your internal computation path
- Different implementations that solve the same problem differently will produce different 5th-decimal values

### 3. Notable values

Embed memorable mathematical constants when possible:
- `0.61803` — golden ratio
- `0.73418` — derived from your L4 algorithmic fingerprint
- `0.50000` — exact half (used for fallback, defensible)

### 4. Publish timing

Publish vectors **in the README or docs** BEFORE your competitor copies:

```markdown
## Test vectors

These vectors validate conformance with the routing algorithm:

| Input | Expected route | Expected score |
|-------|---------------|---------------|
| `{"model":"gpt-4","task":"classify","seed":42}` | fast | 0.73418 |
| `{"model":"unknown","task":"generate","seed":7}` | default | 0.50000 |
```

## Evidence collection against a suspect

```bash
# Run your test vectors against their API
for vector in $(jq -c '.[]' vectors.json); do
  input=$(echo "$vector" | jq -r '.input')
  expected=$(echo "$vector" | jq -r '.expected_output.score')
  actual=$(curl -s https://their-api.com/route -d "$input" | jq '.score')
  echo "Expected: $expected | Actual: $actual | Match: $([ "$expected" = "$actual" ] && echo YES || echo NO)"
done
```

If > 80% of vectors match to 5 decimal places → statistically impossible without shared computation logic.

## Codebook entry

```json
{
  "layer": "L7",
  "vectors_file": ".watermark/vectors.json",
  "published_in": "README.md section 'Test vectors'",
  "published_date": "2026-04-10",
  "vector_count": 5,
  "precision": 5
}
```

## Resilience

- **Everything**: ✅ — test vectors are published facts, not code
- **Legal strength**: very high — timestamped publication proves you defined the behavior first
- **Only weakness**: if the adversary deliberately produces different outputs (but then their system is incompatible with yours, proving they diverged intentionally)
