# REC Phases — Reflect / Execute / Control

Three phases, each with a distinct role, signed certificate output, and timeout+view-change mechanism.

## Phase 0 — Reflect (planning)

### Goal
Produce a canonical plan + per-commis tickets, both signed by the Reflect quorum.

### Agents involved
- 1 Orchestrator (the artist formerly known as "Chef")
- 2 Plan-Validators (Sous-Chef-scope + Sous-Chef-secu from Brigade, reused)

### Flow
1. Orchestrator drafts a plan: `{ tasks: [...], commis_assignments: [...], estimated_cost: X }`
2. Plan is serialized to canonical JSON (sorted keys, no whitespace variation, SHA-256 hashed)
3. Both Plan-Validators independently vote on the **hash**, not the text:
   - APPROVE (hash H is acceptable)
   - DENY (with corrective counter-hash H')
4. If 2-of-2 APPROVE → issue **tickets** (see `leaderless-execute.md`)
5. If disagreement → Orchestrator revises, Phase 0 retries up to 3× → else human escalation

### Certificate out
```json
{
  "phase": "reflect",
  "plan_hash": "sha256:...",
  "signatures": [
    {"signer": "orchestrator", "sig": "..."},
    {"signer": "plan-scope", "sig": "..."},
    {"signer": "plan-secu", "sig": "..."}
  ],
  "timestamp": "2026-04-17T...",
  "tickets": [ /* see leaderless-execute.md */ ]
}
```

### Timeout
**10 minutes** for Phase 0 total. If exceeded → view-change: a backup Orchestrator prompt takes over with the last-agreed partial plan + escalation flag to human.

### Why the hash-canonical voting matters
Two agents asked "is this plan good?" will answer in different words. Paper 2511.15755 showed zero variance requires deterministic inputs to voting. **Vote on the hash**, not the explanation.

## Phase 1 — Execute (parallel work)

### Goal
Commis produce commits that honor their tickets. No Orchestrator involvement required.

### Agents involved
- N Commis (one per task, each holding a signed ticket)
- 1 Contre-chef-inter (auto-approves edits matching ticket scope — see `leaderless-execute.md`)
- 1 Sous-Chef-merge (CI + rebase + push + PR create)
- 1 Maître d'hôtel (post-merge watchdog — inherited from Brigade)

### Flow per commis
1. Commis reads its ticket (signed, scoped)
2. For each intended Edit/Write:
   - Check file path against ticket scope
   - If in scope → contre-chef-inter auto-approves (no round-trip)
   - If outside scope → escalate to Phase 2 Control validators (not Orchestrator) with a mini-quorum (2-of-3)
3. Commits are signed (gitconfig `commit.gpgsign=true` or ephemeral Ed25519 key per commis)
4. Each commit appends an entry to `shared-state.jsonl` with signature
5. When ticket scope complete → Commis sends SendMessage to Sous-Chef-merge "ready for merge"

### Sous-Chef-merge flow
1. Receives "ready for merge" → rebases branch on main
2. Runs quality gates (cargo fmt/clippy/test or language equivalent)
3. Pushes branch
4. `gh pr create` + `gh pr merge --auto --squash`
5. Hands off to Maître d'hôtel for landing watch

### Timeout
**Per task**: 2× the PERT-estimated E(commit) time. If exceeded → view-change: the task is reassigned to a spare commis with the same ticket + incident log.

### Append-only signed log
```jsonl
{"ts":"...","actor":"commis-3","op":"edit","file":"src/router.rs","diff_hash":"sha256:...","sig":"..."}
{"ts":"...","actor":"commis-3","op":"commit","branch":"fix/x","commit_hash":"abc...","sig":"..."}
{"ts":"...","actor":"contre-chef-inter","op":"approve","target":"commis-3","scope_match":"src/router.rs","sig":"..."}
```

Log is append-only (never edited). Rebuild state = fold over the log. Cohérence via ordering, not locks.

## Phase 2 — Control (validation)

### Goal
Independently re-verify each produced artifact against invariants. Issue pass/fail certificates.

### Agents involved
- f+1 Control-Validators (minimum 2 for routine, 3 for sensitive, 4 for critical)
  - Suggested roles: `control-tests`, `control-security`, `control-contracts`, `control-policy`
- Each validator is **independent** (different prompt focus, ideally different model)

### Flow
1. When Sous-Chef-merge signals "PR open, auto-merge enabled", Phase 2 starts
2. Each Control-Validator independently:
   - Pulls the diff
   - Runs their specific check (tests green? no secrets leaked? contracts preserved? policies honored?)
   - Emits APPROVE or DENY with evidence hash
3. Aggregator collects votes:
   - 2-of-f+1 APPROVE → phase certificate emitted, PR can land
   - Any DENY → rollback PR, escalate with evidence to the originating commis for fix
4. Phase certificate appended to shared-state.jsonl

### Certificate out
```json
{
  "phase": "control",
  "pr_number": 212,
  "commit_hash": "abc...",
  "validator_signatures": [
    {"role": "control-tests", "verdict": "APPROVE", "evidence": "sha256:..."},
    {"role": "control-security", "verdict": "APPROVE", "evidence": "sha256:..."},
    {"role": "control-contracts", "verdict": "APPROVE", "evidence": "sha256:..."}
  ],
  "timestamp": "..."
}
```

### Timeout
**5 minutes** per validator. If any validator stalls → view-change: another spawns with the same prompt. **Never block the merge on a single stuck validator.**

## Timeouts summary

| Phase | Timeout | View-change action |
|---|---|---|
| Reflect (global) | 10 min | Spawn backup Orchestrator with partial plan |
| Execute (per task) | 2× PERT E | Reassign task to spare commis |
| Control (per validator) | 5 min | Spawn replacement validator |

**Universal rule**: no phase, no agent, no operation is allowed to hang indefinitely. If the spec doesn't have a timeout, the spec is wrong.

## Phase transition invariants

- **0 → 1**: requires Reflect certificate with all signatures present
- **1 → 2**: requires Execute log with at least one signed commit per active ticket
- **2 → done**: requires Control certificate with 2-of-(f+1) APPROVE

Certificates are public (log) but sigs must be verifiable. The orchestrator-prompt template includes a verifier step at each transition.

## Example sprint flow (grob-s5 redesigned in REC-Quorum)

```
T=0      Phase 0 starts, Orchestrator drafts plan for 6 tasks (B-05..B-09 + R-0 + R-1a/b)
T=3min   Plan hashed, Plan-Validators vote, 2-of-2 APPROVE → certificate emitted
T=3min   6 tickets distributed to 6 commis (opus for R-0/R-1/B-08, sonnet for B-05/06/07/09)
T=3min   Phase 1 begins — leaderless, parallel
         - Contre-chef-inter auto-approves 24 edits across 6 commis (all within scope)
         - 2 edits escalate (one touches Cargo.toml, one touches auth/) → Control mini-quorum
T=90min  All commits done, Sous-Chef-merge creates 6 PRs with auto-merge
T=90min  Phase 2 begins — parallel validation of 6 PRs
T=95min  control-tests: all green, control-security: 5 clean + 1 flag on B-08 auth
         → escalation to human for B-08 only, 5 others land
T=100min Sprint closes with 5-of-6 merged, 1 escalated (vs grob-s5 reality: 2-of-7 merged)
```

No Orchestrator bottleneck. No ccheck-blind inter-agent gate. Signed trail for audit.
