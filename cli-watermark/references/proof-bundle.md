# Proof Bundle — Artifact Generation

> **When to read:** Phase 3, generating the proof artifacts.

---

## What to generate

| File | Content | Public? |
|------|---------|---------|
| `secret.env` | Master secret + derived keys | **NEVER** — private only |
| `codebook.json` | All layers' details (locations, values, techniques) | **NEVER** — private only |
| `PROOF.md` | Commitment hashes + timestamps + verification instructions | ✅ Can be in a private proof repo |
| `vectors.json` | Canonical test vectors (input/output pairs) | ✅ Can be published in README |
| `commitment-{version}.ots` | OpenTimestamps proof file | ✅ Can be shared for verification |
| `VERIFY.md` | Step-by-step forensic verification guide | ✅ Can be shared with legal team |

## codebook.json structure

```json
{
  "project": "grob",
  "version": "1.0.0",
  "created": "2026-04-10T15:00:00+02:00",
  "master_secret_hash": "<SHA256 of MASTER_SECRET — for integrity check, not the secret itself>",
  "layers": {
    "L1": {
      "applied": true,
      "constants": [ ... ]
    },
    "L3": {
      "applied": true,
      "canaries": [ ... ]
    },
    "L4": {
      "applied": true,
      "fingerprints": [ ... ]
    },
    "L5": {
      "applied": true,
      "behavioral_markers": [ ... ]
    },
    "L7": {
      "applied": true,
      "vectors_file": "vectors.json",
      "vector_count": 5
    },
    "L8": {
      "applied": true,
      "commitment": "<hash>",
      "timestamp_proof": "commitment-v1.0.0.ots"
    }
  },
  "resilience_estimate": {
    "rename": "high",
    "restructure": "medium",
    "lang_change": "high",
    "two_rewrites": "high",
    "combined_probability_of_coincidence": "< 1e-12"
  }
}
```

## VERIFY.md template

```markdown
# Watermark Verification Guide — {project}

## For forensic examiners

### Step 1: Obtain the secret
The owner provides MASTER_SECRET under NDA or legal discovery.

### Step 2: Verify the commitment
```bash
# Recompute the commitment
TREE_HASH=$(git ls-tree -r {release_tag} | sha256sum | cut -c1-64)
EXPECTED=$(echo "watermark-commitment:${MASTER_SECRET}:${TREE_HASH}:{timestamp}" | \
  openssl dgst -sha256 | awk '{print $2}')
echo "Match: $([ "$EXPECTED" = "{commitment}" ] && echo YES || echo NO)"

# Verify the timestamp
ots verify commitment-{version}.ots
```

### Step 3: Verify L1 constants
```bash
# Derive the L1 key
L1_KEY=$(echo "L1-constants:${MASTER_SECRET}" | openssl dgst -sha256 | awk '{print $2}')
# Verify each constant matches the derivation
# (see codebook.json for the formula)
```

### Step 4: Verify L4 algorithmic fingerprint
Examine the suspect's code for the specific mathematical constants and algorithmic choices documented in the codebook.

### Step 5: Verify L5 observable behavior
Run the canonical test vectors against the suspect's system. Compare output precision, field ordering, and timing distribution with the owner's production logs.

### Step 6: Statistical argument
The probability that N independent watermark signals all match by coincidence:
- P(L1 match) ≈ 1/10000 per constant × M constants
- P(L4 match) ≈ 1/10000 per fingerprint
- P(L5 match) ≈ 1/100000 per behavioral marker
- P(L7 match) ≈ 1/1000000 per test vector at 5-decimal precision
- Combined: P < 1e-12 (less than one in a trillion)

### Conclusion template
"The examined code exhibits [N] independent markers that are consistent with derivation from the owner's secret key, timestamped [date] via [OpenTimestamps/RFC3161]. The probability of this occurring by independent development is estimated at less than [P]. In the examiner's opinion, this constitutes [strong/moderate] evidence of shared provenance."
```

## Automation script

Generate in `{project}/.watermark/generate-proof.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail

source .watermark/secret.env
VERSION="${1:?Usage: generate-proof.sh <version>}"

# Tree hash
TREE_HASH=$(git ls-tree -r HEAD | sha256sum | cut -c1-64)

# Commitment
TIMESTAMP=$(date -Is)
COMMITMENT=$(echo "watermark-commitment:${MASTER_SECRET}:${TREE_HASH}:${TIMESTAMP}" | \
  openssl dgst -sha256 | awk '{print $2}')

# Record
cat >> .watermark/PROOF.md <<PROOF

## Release ${VERSION}
- Date: ${TIMESTAMP}
- Tree hash: ${TREE_HASH}
- Commitment: ${COMMITMENT}
PROOF

# OpenTimestamps (if available)
if command -v ots >/dev/null 2>&1; then
  echo "$COMMITMENT" > /tmp/wm-commit.txt
  ots stamp /tmp/wm-commit.txt
  mv /tmp/wm-commit.txt.ots ".watermark/commitment-${VERSION}.ots"
  echo "OTS proof: .watermark/commitment-${VERSION}.ots"
fi

echo "Commitment recorded for ${VERSION}: ${COMMITMENT}"
```
