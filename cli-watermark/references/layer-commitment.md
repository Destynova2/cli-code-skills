# L8 — Timestamped Commitment

> **When to read:** Phase 2, applying layer 8. This is the legal proof layer.

---

## Principle

Before each public release, create a cryptographic commitment that proves:
1. You possessed the code at time T
2. You possessed a secret that generates the watermark
3. Both facts are verifiable by a third party

## Commitment construction

```bash
# 1. Hash the source tree
TREE_HASH=$(git ls-tree -r HEAD | sha256sum | cut -c1-64)

# 2. Build the commitment = H(secret || tree_hash || timestamp)
TIMESTAMP=$(date -Is)
COMMITMENT=$(echo "watermark-commitment:${MASTER_SECRET}:${TREE_HASH}:${TIMESTAMP}" | \
  openssl dgst -sha256 | awk '{print $2}')

# 3. Record
echo "---" >> .watermark/PROOF.md
echo "## Release ${VERSION}" >> .watermark/PROOF.md
echo "- Date: ${TIMESTAMP}" >> .watermark/PROOF.md
echo "- Tree hash: ${TREE_HASH}" >> .watermark/PROOF.md  
echo "- Commitment: ${COMMITMENT}" >> .watermark/PROOF.md
```

## Timestamping options

### Option A: OpenTimestamps (Bitcoin blockchain — strongest)

```bash
# Install: pip install opentimestamps-client
echo "$COMMITMENT" > /tmp/commitment.txt
ots stamp /tmp/commitment.txt
mv /tmp/commitment.txt.ots .watermark/commitment-${VERSION}.ots
```

- Anchored to Bitcoin blockchain (tamper-proof)
- Verifiable by anyone: `ots verify commitment-v1.0.0.ots`
- Free, no account needed
- Proof takes ~24h to confirm (Bitcoin block inclusion)

### Option B: RFC 3161 TSA (institutional — fast)

```bash
# Free TSA from freetsa.org
echo "$COMMITMENT" > /tmp/commitment.txt
openssl ts -query -data /tmp/commitment.txt -no_nonce -sha256 -out /tmp/tsq.tsq
curl -s -H "Content-Type: application/timestamp-query" \
  --data-binary @/tmp/tsq.tsq \
  https://freetsa.org/tsr > .watermark/commitment-${VERSION}.tsr
```

- Instant proof
- Verifiable: `openssl ts -verify -data commitment.txt -in commitment.tsr -CAfile freetsa-cacert.pem`
- Accepted by courts (RFC 3161 is a legal standard in the EU)

### Option C: GPG-signed commit in private repo

```bash
# In your private proof repo
echo "$COMMITMENT" > proofs/${VERSION}.txt
git add proofs/${VERSION}.txt
git commit -S -m "commitment: ${VERSION}"
git push
```

- Simple, no external dependency
- Weaker than OTS/TSA (your GPG key's creation date is the anchor, not a third party)

### Recommended: combine A + C

Use OpenTimestamps for the strongest proof AND a GPG-signed private repo for convenience.

## Verification workflow (for a judge/expert)

```
1. Owner reveals MASTER_SECRET
2. Expert derives all layer keys from MASTER_SECRET
3. Expert verifies:
   - L1 constants match the derivation
   - L4 algorithmic choices match the derivation
   - L5 behavioral markers match production logs
   - L7 test vectors produce the expected output
4. Expert verifies the timestamp:
   - ots verify commitment.ots → Bitcoin block T
   - The commitment hash matches H(MASTER_SECRET || tree_hash || timestamp)
5. Conclusion: the owner possessed both the code AND the secret at time T,
   which is before the competitor's first commit
```

## PROOF.md template

```markdown
# Watermark Proof — {project}

This file contains cryptographic commitments for each release.
It does NOT contain the secret. The secret is revealed only in legal proceedings.

## How to verify

1. Obtain the MASTER_SECRET from the owner (via legal discovery or voluntary disclosure)
2. For each release, verify: H("watermark-commitment:" || SECRET || tree_hash || timestamp) == commitment
3. Verify the timestamp: `ots verify commitment-{version}.ots`

## Releases

### v1.0.0
- Date: 2026-04-10T15:00:00+02:00
- Tree hash: a1b2c3d4...
- Commitment: e5f6a7b8...
- Timestamp proof: commitment-v1.0.0.ots (OpenTimestamps)
```
