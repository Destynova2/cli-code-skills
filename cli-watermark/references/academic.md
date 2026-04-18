# Academic References — Code Watermarking Research

> **When to read:** For deeper understanding of the theoretical foundations.

---

## Key papers

### SIR — Semantic Invariant Robust Watermarking (ICLR 2024)

Encodes the watermark in the **semantic embedding space** rather than in token distributions. Two semantically equivalent texts land in the same region of the vector space, so the watermark survives paraphrasing.

**Application to code:** encode your signature in the semantic region of your key functions. An AI rewrite preserves the semantics → preserves the region → detectable by embedding comparison.

### Venkatesan et al. — Graph-Theoretic Software Watermarks (2001)

The watermark is encoded in the **control flow graph** topology. Each node = a basic block, edges = transitions. The watermark is merged into the graph by adding or rearranging edges.

**Why it's still relevant:** the CFG of an algorithm is the most language-agnostic representation. Rust→Go→Python, the CFG stays structurally similar.

### RoSeMary (2025) — ML/Crypto Codesign with ZKP

Combines neural watermarking + zero-knowledge proofs for verification. A third-party arbiter can verify ownership without the owner revealing the signature.

**Application:** ZK-SNARK proof that you possess a secret which generates the watermark, without revealing what that secret is. Prevents the attacker from removing the mark after detection.

### CodeMark-LLM (2025) — Cross-Language via LLM

Uses an LLM to dynamically select semantics-preserving transformations, enabling cross-language watermarking. Previous approaches depended on language-specific features and couldn't scale across languages.

### ICLR 2025 — Multi-LLM-Agent Debate Critical Assessment

Shows that multi-agent debate rarely beats single-agent + chain-of-thought. Relevant because it suggests that **parallel independent analysis** (like our multi-layer approach) beats **adversarial probing** for watermark detection.

## Key findings across the literature

1. **Syntactic watermarks are dead** — simple transformations (renaming, dead code insertion) drop detection below 50%
2. **Semantic watermarks survive paraphrasing** — embedding-space methods are robust to rewording
3. **CFG watermarks survive language change** — the graph topology is the most invariant property
4. **ZKP enables public verification without secret exposure** — the forensic model of the future
5. **Combination is key** — no single technique is unbreakable; layered approaches with independent signals make coincidence astronomically improbable

## Recommended stack (based on 2025 state of the art)

| Layer | Technique | Source |
|-------|-----------|--------|
| Code-level | CFG topology (Venkatesan) + algorithmic fingerprint | Classic + our L4 |
| Behavior-level | Observable output fingerprint | Our L5 |
| Proof-level | OpenTimestamps + ZKP commitment | RoSeMary + our L8 |
| Legal-level | Published test vectors + production logs | Our L7 |

This is the strongest known combination as of 2025.
