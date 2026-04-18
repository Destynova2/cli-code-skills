# BFT Primitives — What to reuse, what not to re-implement

REC-Quorum is a dispatch pattern, not a consensus protocol. Do **not** write your own BFT from scratch. Pick from existing, battle-tested primitives, and wire them.

## Consensus protocols — pick one

| Protocol | Best for | Complexity | Notes |
|---|---|---|---|
| **PBFT** | reference implementation, < 10 nodes | O(n²) per round | classic 1999, foundation of everything |
| **HotStuff / HotStuff-2** | linear scalability, pipelining | O(n) per round | used by Diem, Aptos. Responsive (no hard timeouts). **Best default for REC-Q.** |
| **Mir-BFT / Arma** | horizontal scaling, multi-leader | O(n) amortized | several leaders in parallel, no single bottleneck |
| **Mercury** | low-latency wide-area | dual-mode (fast/slow) | dual resilience threshold — compact quorum when healthy, full on suspicion. Sub-second finality claimed. [paper](https://arxiv.org/abs/2305.15000) |
| **BFTBrain** | variable workload | RL-driven protocol swap | contextual multi-armed bandit picks PBFT / Zyzzyva / HotStuff per workload. +18-119% throughput. [NSDI 2025](https://arxiv.org/abs/2408.06432) |
| **Raft** | crash-fault only (not BFT) | O(n) | simpler than BFT. Use only if threat model excludes Byzantine faults. |

### Decision tree

1. Small team (≤ 5 agents), dev use case → **Raft** (no BFT needed, saves complexity)
2. Medium team (6-20), BFT required → **HotStuff-lite**
3. Large team (> 20) OR high parallelism → **Mir-BFT**
4. Latency-sensitive, WAN deployment → **Mercury**
5. Workload highly variable → **BFTBrain** (RL-driven)

Default recommendation for REC-Quorum generation: **HotStuff-lite**. Most deployments don't need the complexity of Mir or BFTBrain.

## Threshold signatures

For certificate signing at phase boundaries. Never sign per-operation — batch at phase exit only.

| Scheme | Sig size | Aggregatable | Battle-tested | Libs |
|---|---|---|---|---|
| **BLS12-381** | 48 bytes (G1) | YES (additive) | widely used (Eth2, Filecoin) | `blst` (C), `ark-bls12-381` (Rust), `py_ecc` (Python) |
| **Schnorr MuSig2** | 64 bytes | YES (multi-sig) | Bitcoin Taproot era | `secp256k1-zkp`, `musig2-rs` |
| **Ed25519 (threshold via FROST)** | 64 bytes per share | YES (via MPC) | newer but auditable | `frost-ed25519` |

### Decision

Default: **BLS12-381** for aggregation speed and sig size. Use Schnorr if interfacing with Bitcoin-style infrastructure.

### Anti-pattern
Do NOT use one ECDSA signature per node and verify n of them individually. That's O(n) verification per certificate. Threshold sigs give O(1) verification.

## Membership & gossip

For keeping track of which nodes are alive.

| Protocol | Best for | Notes |
|---|---|---|
| **SWIM** | small-to-medium clusters | fast failure detection, used by Consul |
| **HashiCorp memberlist** | Go deployments | SWIM implementation, battle-tested |
| **Serf** | higher-level on memberlist | event bus, user-defined queries |
| **Raft group** | if you already have Raft for consensus | reuse the same leader election |

For REC-Quorum inside a tmux + local processes setup, you don't need gossip — membership is static (declared in tmuxinator config). This section matters if you scale to multi-host.

## View-change / leader rotation

Every leader-based protocol has a view-change. Implementation varies:

- **PBFT**: explicit view-change with a 3-phase protocol (expensive)
- **HotStuff**: implicit view-change via timeout → next view starts, same flow
- **Mir-BFT**: multi-leaders, no view-change needed (rotation by design)

For REC-Q, leverage your chosen protocol's mechanism — don't invent one. If HotStuff, use HotStuff's view-change. The REC-Q layer only adds a timer per phase that triggers the library's view-change API.

## Append-only log — signed and ordered

For the shared-state.jsonl audit trail:

- **Each entry signed** by the actor (at minimum a hash of the previous entry + current payload)
- **Ordered** by hash-chaining (entry N includes hash of entry N-1)
- **Append-only** — no edits, no rewrites. Rebuild state = fold over log.
- **Verifiable** — any reader can replay from genesis and verify every signature

Implementation: plain JSONL file + Ed25519 signatures + SHA-256 hash chain. No need for a full database.

Example entry:
```json
{
  "seq": 247,
  "ts": "2026-04-17T10:00:00Z",
  "actor": "commis-3",
  "op": "commit",
  "payload": {"branch": "fix/x", "commit_hash": "abc123"},
  "prev_hash": "sha256:...",
  "entry_hash": "sha256:...",
  "sig": "ed25519:..."
}
```

## What REC-Quorum generates vs what it expects pre-existing

**Generates**:
- Orchestrator prompt (Phase 0 logic)
- N validator prompts (Phase 2)
- Contre-chef-inter prompt (Phase 1 ticket verification)
- Petri net / BPMN diagram
- shared-state.jsonl bootstrap (genesis entry)
- tmuxinator config

**Expects pre-existing** (pointers, not generated):
- A BFT consensus library (HotStuff-lite recommended)
- A threshold signature library (BLS preferred)
- A Petri analysis tool (Renew or WoPeD)

If the project lacks these, the skill generates a **stub** that panics loudly on first use, with instructions to install. Do NOT silently fall back to unsigned operations.

## Further reading

- [PBFT original paper (Castro & Liskov, 1999)](https://pmg.csail.mit.edu/papers/osdi99.pdf)
- [HotStuff paper (2019)](https://arxiv.org/abs/1803.05069)
- [Mir-BFT paper](https://arxiv.org/abs/1906.05552)
- [BFTBrain NSDI 2025](https://www.usenix.org/system/files/nsdi25-wu-chenyuan.pdf)
- [Mercury Middleware 2024](https://dl.acm.org/doi/10.1145/3652892.3700756)
- [BLS signatures specification (IETF draft)](https://datatracker.ietf.org/doc/draft-irtf-cfrg-bls-signature/)
- [FROST threshold Ed25519](https://datatracker.ietf.org/doc/draft-irtf-cfrg-frost/)
