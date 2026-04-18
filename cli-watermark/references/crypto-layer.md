# Crypto Layer — Secret Generation & Key Derivation

> **When to read:** Phase 1 — generating the master secret and per-layer keys.

---

## Master secret

```bash
# Generate once per project. Store in .watermark/secret.env (GITIGNORED)
MASTER_SECRET=$(openssl rand -hex 32)
echo "MASTER_SECRET=${MASTER_SECRET}" > .watermark/secret.env
echo "CREATED=$(date -Is)" >> .watermark/secret.env
chmod 600 .watermark/secret.env
```

## Per-layer key derivation

Each layer gets its own key derived from the master via HKDF-like construction:

```bash
derive_key() {
  local layer="$1"
  local secret="$2"
  echo "${layer}:${secret}" | openssl dgst -sha256 | awk '{print $2}'
}

L1_KEY=$(derive_key "L1-constants" "$MASTER_SECRET")
L2_KEY=$(derive_key "L2-synonyms" "$MASTER_SECRET")
L3_KEY=$(derive_key "L3-canaries" "$MASTER_SECRET")
L4_KEY=$(derive_key "L4-algorithm" "$MASTER_SECRET")
L5_KEY=$(derive_key "L5-behavior" "$MASTER_SECRET")
```

## Constant generation from key

```bash
# Generate N hex constants from a layer key
generate_constants() {
  local key="$1"
  local count="$2"
  for i in $(seq 1 "$count"); do
    echo "${key}:const:${i}" | openssl dgst -sha256 | awk '{print $2}' | cut -c1-8
  done
}

# Example: generate 5 constants for L1
generate_constants "$L1_KEY" 5
# Output: 1a4f2b7e, 0d3c8e91, ...
```

## Per-release versioning

Each release gets a new derivation:

```bash
RELEASE_KEY=$(derive_key "release:v${VERSION}:$(date -Is)" "$MASTER_SECRET")
```

This lets you prove WHICH version's constants belong to WHICH release.

## Storage layout

```
{project}/.watermark/           # GITIGNORED — never commit
├── secret.env                  # master secret
├── codebook.json               # all layers' details
├── vectors.json                # test vectors (CAN be published)
├── PROOF.md                    # commitment hashes (CAN be published)
├── commitment.ots              # OpenTimestamps proof
└── releases/
    ├── v1.0.0.json             # per-release codebook snapshot
    └── v1.1.0.json
```

## .gitignore entry

```
# Watermark secrets — NEVER commit
.watermark/secret.env
.watermark/codebook.json
.watermark/releases/
```

Note: `PROOF.md`, `vectors.json`, and `commitment.ots` CAN be committed to a private proof repo.
