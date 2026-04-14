# Simplified Model — Stigmergy, Apoptosis, DNA Repair

> **When to read:** Phase 0 — to decide which model to use (simplified or full brigade).

---

## Principle: nature does not coordinate, it lets things emerge

The Brigade de Cuisine model (Chef + 3 Sous-Chefs + Merge + N Commis) is justified for regulated projects (XL). For tiers S/M/L, nature offers a simpler model: **no central coordinator, agents self-organize via the shared environment.**

```
Full brigade (XL):                  Stigmergy (S/M/L):
  Chef coordinates                   shared-state.md coordinates
  3 Sous-Chefs vote                  1 Sous-Chef (3 lenses) reviews
  1 Sous-Chef                  Commis merge themselves
  N Commis execute                   N Commis self-assign

  6+ agents                          2+ agents (1 sous-chef + N commis)
  Inter-agent messages               Read/write shared-state
  Voting protocol                    Sous-chef internal decision
  Chef = SPOF                        No SPOF
```

---

## Decision: which model to use

| Tier | Model | Agents | Reason |
|------|-------|--------|--------|
| **S** (1-2 tasks) | No brigade | 1 commis alone | One cook does not need a chef |
| **M** (3-4 tasks) | **Stigmergy** | 1 sous-chef + N commis | Self-organization is enough |
| **L** (5+ tasks) | **Stigmergy** | 1 sous-chef + N commis | Same model, more commis |
| **XL** (regulated) | **Full brigade** | Chef + 3×3 sous-chefs + N commis | Audit trail, compliance, traceability |

---

## The 5 patterns of the simplified model

### 1. Boids — 3 rules for the commis (replaces the runtime Chef)

Each commis follows 3 rules. No Chef assigning or watching.

```
SEPARATION: do not touch a file locked by another commis
            (check shared-state.md "Locks" section)

ALIGNMENT:  follow the same conventions — commit style (cli-git-conventional),
            project language, branch naming

COHESION:   converge toward the sprint goal
            (check shared-state.md "PERT" section for progress)
```

The Chef no longer exists at runtime. It generates the files in Phase 0, then disappears. The commis coordinate via shared-state.md.

### 2. Quorum Sensing — 1 Sous-Chef, 3 lenses (replaces 3 voting Sous-Chefs)

One review agent applies 3 lenses sequentially. No vote, no resolution rounds, no inter-agent consensus.

```
Sous-Chef receives a diff:

  Lens 1 — SCOPE: is the change inside the task perimeter?
    IF out of scope → DENY("out of perimeter: touches {file} not assigned")

  Lens 2 — SECURITY: exposed secrets? unverified deps? eval()?
    IF security risk → sensitive zone: ESCALATE to the patron
    IF security risk → normal zone: DENY("secret detected in {file}:L{line}")

  Lens 3 — QUALITY: tests pass? clean code? no regression?
    IF low quality → DENY("missing test for {function}")

  All 3 lenses pass → APPROVE
```

**When to keep 3 separate sous-chefs:**
- Tier XL (regulated)
- Diff > 200 lines
- Sensitive zone (3/3 unanimity required)
- The user explicitly asks for it

### 3. DNA Repair — adaptive quality gates (replaces systematic gates)

Gates are not the same for every diff. Like the cell: constant proofreading, but checkpoint only when there's a problem.

```
diff_size < 50 lines AND normal zone:
  → PROOFREADING: the commis auto-verifies (test + lint)
  → Sous-Chef: quick check (30s, 1 lens only — quality)
  → No audit skill

diff_size 50-200 lines OR hot zone:
  → MISMATCH REPAIR: the sous-chef runs /cli-audit-code
  → 1 skill, not 4

diff_size > 200 lines OR sensitive zone:
  → CHECKPOINT: the sous-chef runs the full battery
  → /cli-audit-code + /cli-audit-shell (if .sh) + /cli-audit-drift (if CONTRACTS.md)
```

**Result:** 60-70% of merges (small diffs in a normal zone) trigger no audit skill. The sous-chef validates them in 30 seconds.

### 4. Reaction-Diffusion — self-organized task pool (driven by the PERT)

The PERT is computed in Phase 0.5 (`references/pert-computation.md`) and written to shared-state.md with a priority per plat. The commis then help themselves from the pool — no Chef scheduling loop.

**Two numbers per plat:**

- **Readiness** ∈ {0, 1}: 1 iff all PERT predecessors are `Envoye`. A plat only becomes pickable when all its inbound arrows in the PERT are green. There is no partial readiness — either a successor can start or it cannot.
- **Priority** = longest path from the plat to `done` (cf. pert-computation.md §4). Static, computed once. Critical-path plats have the highest priority by construction.

```markdown
## Task pool

| Plat | Ready | Priority | Slack | Write-set | Commis |
|------|-------|----------|-------|-----------|--------|
| auth-refactor | 1 | 12.0 | 0 | src/auth/* | commis-1 |
| schema-migrate | 1 | 8.0 | 2 | migrations/* | commis-2 |
| api-endpoints | 0 | 8.0 | 0 | src/api/*, src/auth/mod.rs | - |
| docs-update | 0 | 1.0 | 4 | docs/* | - |
```

**Dispatch rules (file-exclusion + critical-path-first):**

1. A free commis scans the ready rows (`Ready = 1`).
2. It filters out any row whose `Write-set` intersects the `Write-set` of a row already assigned (`Commis ≠ -`). This is the "don't tangle the brushes" rule — the commis never fights another commis for a file.
3. It picks the highest-remaining priority. Ties → smaller slack, then smaller E.
4. It writes its name into `Commis` (row lock). First come, first served if two commis race — the second one re-runs step 2 and picks the next candidate.
5. When a plat merges (`Envoye`), the commis updates `Ready` on every successor whose predecessors are now all green. The pool re-sorts itself, no Chef intervention.

**Why priority and not just readiness?** Picking the "most ready" task starves the critical path: the chef ends up using its best commis on slack-rich plats while critical work sits in the queue. Sorting by longest-remaining-path guarantees that scarce commis-hours are spent on the tasks that set the makespan — the definition of "aller le plus vite".

### 5. Apoptosis — self-termination of failing commis (replaces Patch Bankruptcy + Divergence Detection)

A failing commis terminates cleanly, no Chef intervention.

```
IF same_diff_rejected >= 2:
  → APOPTOSIS (wrong approach)
  → git revert to last-good-state
  → release all locks
  → write to shared-state: "TASK {x}: APOPTOSIS by commis-{n}, reason: {motive}"
  → the task returns to the pool with readiness = 1.0
  → another commis will pick it up with a different approach

IF idle > 30min with no progress:
  → APOPTOSIS (stuck)
  → same procedure

IF regression detected (file that passed tests no longer passes):
  → git revert of the last edit
  → retry 1 time
  → if still failing → APOPTOSIS
```

**Like Kubernetes:** the pod (commis) is disposable. The workload (task) is what matters. You don't debug a failing commis, you replace it.

---

## Runtime architecture of the simplified model

```
┌─────────────────────────────────────────────┐
│           shared-state.md                    │
│  (the only coordination medium)              │
│                                              │
│  Sections:                                   │
│  - Task pool (readiness scores)              │
│  - Active locks                              │
│  - Completion tokens                         │
│  - Sensitive zones                           │
│  - Commis status (ACTIVE/APOPTOSIS)          │
│  - Backlog for the next sprint               │
└─────────┬───────────┬───────────┬────────────┘
          │           │           │
     ┌────▼────┐ ┌────▼────┐ ┌───▼─────┐
     │Commis 1 │ │Commis 2 │ │Commis N │
     │         │ │         │ │         │
     │ Boids:  │ │ Boids:  │ │ Boids:  │
     │ separ.  │ │ separ.  │ │ separ.  │
     │ align.  │ │ align.  │ │ align.  │
     │ cohes.  │ │ cohes.  │ │ cohes.  │
     │         │ │         │ │         │
     │ Apopt:  │ │ Apopt:  │ │ Apopt:  │
     │ 2 reject│ │ 2 reject│ │ 2 reject│
     │ = die   │ │ = die   │ │ = die   │
     └────┬────┘ └────┬────┘ └────┬────┘
          │           │           │
          └─────┬─────┘           │
                │ "Pret !"        │
          ┌─────▼─────┐           │
          │ Sous-Chef  │◄──────────┘
          │ (3 lenses) │
          │            │
          │ scope      │
          │ security   │
          │ quality    │
          │            │
          │ DNA repair:│
          │ small=skip │
          │ medium=1   │
          │ large=full │
          └────────────┘
```

---

## Comparison

| Aspect | Full brigade | Stigmergy |
|--------|--------------|-----------|
| Runtime agents | 6+ (Chef+3SC+Merge+N) | 2+ (1SC+N) |
| Coordination | Inter-agent messages | File read/write |
| Point of failure | Chef = SPOF | No SPOF |
| Token cost | High (Chef thinks + 3 votes) | Low (1 review + N commis) |
| Latency per merge | High (vote → resolution → merge) | Low (review → merge) |
| Audit trail | Explicit (votes, decisions) | Implicit (shared-state.md history, git log) |
| Regulatory compliance | Yes (vote traceability) | No (no auditable vote) |
| Scalability | Limited by the Chef | Unlimited (add more commis) |
