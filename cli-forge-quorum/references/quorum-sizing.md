# Quorum Sizing — Adaptive N by Criticality

## The classifier

Every task entering REC-Quorum is classified into one of four criticality levels, which maps to quorum size N and fault tolerance.

| Criticality | N | Tolerates | Example tasks | LLM cost multiplier |
|---|---|---|---|---|
| **trivial** | 1 | 0 faults | rename variable, format comment, doc typo fix | 1× |
| **routine** | 3 | 1 crash-fault | standard feature impl, bug fix, refactor | 3× |
| **sensitive** | 5 | 1 Byzantine fault | security-adjacent code, dependency bump, API contract change | 5× |
| **critical** | 7 | 2 Byzantine faults | auth logic, crypto, financial transactions, safety systems | 7× |

## Classification rules (in order, first match wins)

1. **File path match**:
   - `**/Cargo.toml`, `**/package.json`, `**/*.lock` → **sensitive** (deps are attack vectors)
   - `.github/workflows/**`, `Dockerfile*` → **sensitive** (CI/supply chain)
   - `src/auth/**`, `src/crypto/**`, `src/security/**` → **critical**
   - `src/payment/**`, `src/billing/**` → **critical**
   - `docs/**`, `README.md`, `*.md` → **trivial**
   - `tests/**` → **routine**

2. **Content match** (if file path ambiguous):
   - Contains `unsafe`, `extern "C"`, `#[no_mangle]` → **critical**
   - Contains `password`, `secret`, `api_key`, `token` literals → **sensitive**
   - Contains SQL raw queries, shell command substitution → **sensitive**
   - Pure refactor (no semantic change expected) → **routine**

3. **Diff size**:
   - > 500 lines added → bump one level up (too big to trust one validator)
   - > 2000 lines → bump two levels

4. **Commit message hints**:
   - `BREAKING:`, `security:`, `fix(cve-*)` → **sensitive** minimum
   - `chore:`, `docs:`, `style:` → usually **trivial** (but verify via path)

5. **Default if nothing matches**: **routine**

## LLM cost model

A sprint with task mix `T trivial, R routine, S sensitive, C critical`:

```
base_cost_per_task = cost(Reflect) + cost(Execute_1_commis) + cost(Control_per_validator)
                   ≈ $0.05 + $0.30 + $0.05 per task (Opus-scale, 50k input / 10k output)

rec_q_cost = T × base × 1
           + R × base × 3   (Execute stays 1 commis; Control scales)
           + S × base × 5
           + C × base × 7

Note: only Control phase scales with N. Execute remains 1 commis per task.
Reflect is fixed overhead per sprint (~$0.50 for all tasks combined).
```

### Example: grob-s5 redesigned under REC-Quorum

Task mix: 6 routine + 1 critical = R=6, C=1

- Reflect (sprint-wide): ~$0.50
- Execute (6 + 1 commis): 7 × $0.30 = $2.10
- Control: 6 × 3 × $0.05 + 1 × 7 × $0.05 = $0.90 + $0.35 = $1.25
- **Total: $3.85 vs Brigade at ~$2.10 = 1.8× multiplier**

For context: one 45-min Opus stall costs ~$8-12 of wasted thinking tokens + human time. REC-Q pays for itself on the first avoided stall.

## Heterogeneous quorums — the "1 strong + N-1 watchdogs" trick

For cost-sensitive critical tasks, use **asymmetric slot strength**:

- 1 slot uses the strong model (Opus 4.7) for the main reasoning
- N-1 slots use cheap models (Haiku 4.5) as independent watchdogs

The watchdogs verify specific invariants:
- **watchdog-tests**: "does `cargo test` pass?"
- **watchdog-diff-size**: "is the diff within claimed scope?"
- **watchdog-secrets**: "no secret strings in diff?"
- **watchdog-contract**: "public API signatures preserved?"

These questions are mechanical and Haiku handles them fine. You get the safety of N=7 at roughly the cost of N=2 strong + 5 weak = 3-4× instead of 7×.

## When to override the classifier

The operator can manually set criticality:

- **Escalate a routine to sensitive**: for high-stakes pre-release sprints where even small bugs hurt
- **Downgrade a sensitive to routine**: for prototypes in a sandbox with no production exposure (document the risk in shared-state)

Overrides are logged (who decided, why) in the Reflect certificate. Audit trail preserved.

## Admission control — refuse if quorum unreachable

If a task classifies to N=5 but only 3 validator slots are healthy (due to prior crashes/timeouts), the orchestrator **rejects** the task with backpressure:

```
"Task T classified as sensitive (N=5), only 3 healthy validator slots available.
 Reject with backpressure. Operator action: wait for view-change recovery OR
 explicitly downgrade to routine with justification."
```

Never execute with lower N than the classifier demands without an explicit, signed downgrade certificate.

## BFT formalism for the curious

The bound `N = 3f + 1` comes from PBFT — to tolerate f Byzantine faults, you need at least 3f+1 total nodes.

| N | f (Byzantine tolerance) | f (crash-only) |
|---|---|---|
| 1 | 0 | 0 |
| 3 | 0 | 1 (crash) |
| 4 | 1 | 1 |
| 5 | 1 | 2 |
| 7 | 2 | 3 |

REC-Quorum picks odd N for simpler majority computation. N=4 is skipped (tolerates only 1 Byzantine, same as N=5, but even numbers complicate tie-breaking).

Mercury's dual-mode trick: when no fault detected, use smaller quorum (floor(N/2)+1) for speed; escalate to full N only on disagreement. We simplify in REC-Quorum by starting at classified N; worth re-introducing if latency becomes an issue.
