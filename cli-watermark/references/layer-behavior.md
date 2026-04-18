# L5 — Observable Behavior Fingerprint

> **When to read:** Phase 2, applying layer 5. This is the strongest layer overall.

---

## Principle

Encode your fingerprint in externally measurable behavior — not in the code itself. The behavior is what the binary produces when run. Any rewrite in any language that copies your logic copies your behavior.

Your production logs, timestamped before the competitor's first commit, prove the behavior is yours.

## Techniques

### 1. Timing padding

Add a deterministic micro-delay derived from request content:

```rust
let delay_us = hmac_sha256(SECRET, &request_id)[0..2] as u16 % JITTER_WINDOW;
tokio::time::sleep(Duration::from_micros(delay_us as u64)).await;
```

- The delay is small enough to be invisible to users (< 1ms)
- The distribution over N requests is unique to your secret
- Measurable by anyone with access to the API

### 2. JSON field ordering

Produce JSON with a non-alphabetical, non-insertion-order field sequence:

```json
{"model": "...", "id": "...", "ts": "...", "score": 0.42}
```

This specific ordering (model → id → ts → score) is improbable and stable. Most serializers use either alphabetical or struct-field order — neither produces this exact sequence unless deliberately configured.

### 3. Response header fingerprint

```
x-grob-rid: req_1A4F2B7E3C...
```

- Fixed length (always 18 chars after prefix)
- Non-standard prefix
- Deterministic from the request content

### 4. Output precision fingerprint

Choose a specific decimal precision for numeric outputs:

```json
{"score": 0.61803}
```

- 5 decimal places (not 2, not 8)
- The value itself may encode the golden ratio
- Any implementation that reproduces this exact precision on your test vectors shares your computation path

### 5. Error message fingerprint

Use specific, unusual phrasing in error messages:

```
"model not routable: no upstream satisfies the constraint set"
```

vs the more common:
```
"model not found"
```

The exact phrasing survives any rewrite that preserves error messages.

## Evidence collection

When you suspect copying:

```bash
# 1. Send your canonical test vectors to their API
curl -s https://their-api.com/route -d '{"model":"gpt-4","task":"classify","seed":42}' | jq .

# 2. Compare field ordering
# 3. Compare numeric precision
# 4. Compare response headers
# 5. Measure timing distribution over 1000 requests

# Your evidence: their API produces the same fingerprint as yours
# Your proof: your production logs show this fingerprint months before their launch
```

## Codebook entry

```json
{
  "layer": "L5",
  "behavioral_markers": [
    {
      "type": "field_ordering",
      "pattern": ["model", "id", "ts", "score"],
      "endpoint": "/route",
      "first_seen_in_prod": "2026-03-15T14:32:00Z"
    },
    {
      "type": "precision",
      "field": "score",
      "decimals": 5,
      "notable_value": 0.61803
    },
    {
      "type": "timing_padding",
      "jitter_window_us": 500,
      "derivation": "HMAC-SHA256(L5_KEY, request_id)[0:2] mod 500"
    }
  ]
}
```

## Resilience

- **Everything**: ✅ — the behavior is what the binary produces, not what the source looks like
- **Only weakness**: if the adversary deliberately changes all observable behavior (different field ordering, different precision, different headers). But that requires knowing what to change — and if they change the algorithm's output, they break compatibility.

**This layer + production logs = strongest evidence in court.**
