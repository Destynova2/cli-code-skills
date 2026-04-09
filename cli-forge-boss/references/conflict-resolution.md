# Conflict Resolution — Decision Tree

> Sources: Anthropic C compiler build, Claude Code Agent Teams docs, AlphaCode, MetaGPT, Addy Osmani "Code Agent Orchestra"

## Principles

1. **Prevention > Detection > Resolution.** Invest 80% in prevention, 15% in detection, 5% in resolution.
2. **The Sous-Chef handles conflicts, never the Commis.** A commis that rebases alone can break someone else's work.
3. **3 attempts max.** After 3 failed resolutions, escalate to the Chef who decides (reassign, sequence, or merge manually).

## Decision tree

```
A commis announces "Pret !"
  │
  ▼
The Sous-Chef attempts the merge
  │
  ├─ OK (no conflict) ─────────── Merge + CI + "Plat envoye"
  │
  └─ CONFLICT detected
       │
       ▼
     What kind of conflict?
       │
       ├─ Different file, same line (e.g., Cargo.toml dependencies)
       │    │
       │    ▼
       │  AUTO RESOLUTION: merge both additions
       │  (the Sous-Chef runs `git checkout --theirs` or merges manually)
       │  → Re-run CI → if OK → "Plat envoye"
       │
       ├─ Same file, same function (semantic conflict)
       │    │
       │    ▼
       │  The Sous-Chef asks the 2 commis involved:
       │  "What is the intent of your change on fn X?"
       │  → Compare the intents
       │    │
       │    ├─ Compatible intents → the Sous-Chef merges
       │    │
       │    └─ Contradictory intents → escalate to the Chef
       │         │
       │         ▼
       │       The Chef decides:
       │         ├─ Keep approach A (reassign B)
       │         ├─ Keep approach B (reassign A)
       │         ├─ Merge (new spec)
       │         └─ Sequence (A first, then B adapts)
       │
       └─ Shared config file (ci.yml, Cargo.toml, package.json)
            │
            ▼
          RULE G15: Sequence the merges
          1. Merge the oldest PR first
          2. Rebase the next one onto the result
          3. Re-run CI on the rebased PR
          4. Merge if CI is green
```

## Sous-Chef protocol (before every merge)

```
1. git fetch origin
2. git log --oneline origin/{base}..{worker_branch} → list touched files
3. Compare with the files of the PRs already merged in this session
4. IF overlap detected:
     → Try a local merge (without push)
     → IF conflict:
         → Read the diff from both sides
         → IF trivial resolution (non-conflicting additions): auto-resolve
         → OTHERWISE: ask the commis involved via SendMessage
5. Run CI locally (cargo test / npm test / etc.)
6. IF green: push + "Plat envoye"
7. IF red: identify the regression, send it back to the commis
```

## Attempt counter

| Attempt | Action |
|---------|--------|
| 1 | Sous-Chef tries an auto merge (trivial conflicts) |
| 2 | Sous-Chef asks the commis to explain their intent, merges |
| 3 | Escalate to the Chef: human decision or reassignment |

After attempt 3, the Chef may:
- **Reassign**: one commis takes both tasks
- **Sequence**: turn parallel tasks into PERT dependencies
- **Split**: refactor the conflicting file into separate modules

## Reinforced prevention (Phase 0)

### Cycle detection on the PERT (Tarjan — inspired by cli-audit-tangle)

Before launching the commis, verify that the dependency graph is a DAG:

```
1. Build the graph: task_A → task_B if A must finish before B
2. Apply Tarjan SCC on the graph
3. If any SCC with > 1 node → CYCLE DETECTED → do not launch

Forbidden cycle example:
  task_1 (auth) depends on task_2 (database schema)
  task_2 (database) depends on task_1 (auth models)
  → SCC = {task_1, task_2} → CYCLE

Resolution:
  - Merge the 2 tasks into 1 (same commis)
  - OR break the dependency (interface/mock)
  - OR sequence explicitly (A then B, no parallelism)
```

### Forward-only rule

Dependencies between commis can only go ONE way:
- Commis A may depend on the result of Commis B
- But B can NOT depend on A in return
- If bidirectional → merge into a single commis

### God-commis detection (Fire ants)

A commis touching too many files = an ant gripping 12 neighbors = SPOF.

```
For each commis, count:
  files_touched = number of files in its task
  total_files = total number of files touched across all commis

If files_touched / total_files > 0.5 → GOD-COMMIS
  → Split the task across 2 commis
  → OR reassign some files to other commis

If a commis has W/W overlap with > 2 other commis → HUB
  → Sequence its merges last
  → OR turn it into a "base commis" the others depend on
```

### Coupling matrix

Before the coup de feu, the Chef builds a matrix:

```
          file_a   file_b   file_c   Cargo.toml
task_1      W        R        -        W
task_2      -        W        W        W
task_3      R        -        W        R
```

- **W/W in the same column** = guaranteed conflict → assign to the same commis OR sequence
- **R/W** = low risk, acceptable in parallel
- **R/R** = no risk

### "Hot" files (always in conflict)

Some files get touched by almost every task:
- `Cargo.toml` / `package.json` (new deps)
- `ci.yml` / `.github/workflows/` (new jobs)
- `mod.rs` / `index.ts` (re-exports)
- `CHANGELOG.md`

**Rule**: these files are handled by the Sous-Chef AFTER all the merges. Commis never touch them directly — they list their needs in shared-state.md ("Need: add dep X in Cargo.toml") and the Sous-Chef applies them in a single final pass.

## File locking via shared-state.md

Inspired by Anthropic's pattern (C compiler build):

```markdown
## Active locks

| File | Commis | Since | Reason |
|------|--------|-------|--------|
| src/auth/mod.rs | commis-1 | 14:32 | refactoring auth flow |
```

**Rules:**
1. A commis that wants to edit a file checks the locks first
2. If locked by another → SendMessage to the Sous-Chef: "Need to edit {file}, locked by {commis}"
3. The Sous-Chef decides: wait, reassign, or authorize (if the zones in the file are different)
4. The commis releases the lock when saying "Pret !"

---

## Stigmergy — completion tokens (inspired by leafcutter ants, cli-forge-pipeline)

Commis do not communicate directly with each other. They drop **tokens** into shared-state.md when a subtask is done. Dependent commis wait for the token, not a message from the Chef.

```markdown
## Completion tokens

| Token | Commis | Timestamp | Consumable by |
|-------|--------|-----------|---------------|
| schema-db-ready | commis-1 | 14:45 | commis-2, commis-3 |
| api-endpoints-defined | commis-2 | 15:10 | commis-4 |
```

**Rules:**
1. A commis drops a token when its subtask is done (not when its full plat is ready)
2. A dependent commis polls for tokens before starting its dependent part
3. The Chef does NOT intervene in token synchronization — this is stigmergy
4. If a token doesn't arrive within the expected delay → the dependent commis alerts the Sous-Chef

---

## Patch Bankruptcy (inspired by cli-forge-infra)

After 2 gate rejections on the same diff, the probability that the approach is correct is 25%.

```
Attempt 1: gate rejects → commis fixes
Attempt 2: gate rejects again → MANDATORY STOP
  → The commis does NOT retry
  → The Chef reviews: wrong approach? wrong spec? wrong commis?
  → Options:
    a) Reassign to another commis (different approach)
    b) Redefine the spec (the Chef clarifies intent)
    c) Merge with another plat (the task is poorly split)
    d) Escalate to the patron (human)
```

**Never 3 attempts on the same approach.** It is statistically irrational (P=87.5% the approach is wrong).

---

## Divergence detection (inspired by Physarum / cli-forge-pipeline)

If a commis stalls (no progress > 30min) or regresses (gate approval → gate rejection on the same file):

```
IF commis.idle_time > 30min:
  → Chef alerts: "Commis {X} inactive for 30min on {plat}"
  → Options: restart, reassign, escalate

IF commis.regression_detected (gate approved on file A, then rejected after new edit):
  → Sous-Chef flags: "Regression on {file} — edit {hash} broke what was working"
  → Commis must `git revert` before continuing

IF same_gate.rejects(commis, same_diff) >= 2:
  → Patch Bankruptcy (see above)
```

---

## Convergence and early-stop (inspired by cli-cycle Phoenix)

The sprint stops automatically when:

| Condition | Name | Action |
|-----------|------|--------|
| Every plat in the PERT is "Envoye" | **COMPLETE** | Normal service report |
| Quality score >= target AND 0 Tier 3 remaining | **CONVERGED** | Stop even if minor plats remain |
| Gate rejection rate > 20% over the last 5 diffs | **MISALIGNED** | Stop, replan — sous-chefs and commis disagree |
| No merges in > 1h with active commis | **STUCK** | Escalate to the Chef, NFD diagnostic (check communication before logic) |
| Tier 3 created by fix > Tier 3 resolved | **DIVERGING** | Immediate stop, present the state to the patron |

---

## Sprint Health Scorecard (inspired by the cli-audit-* scoring frameworks)

To include in the final Service Report:

```markdown
## Sprint health

| Metric | Value | Threshold | Status |
|--------|-------|-----------|--------|
| Gate approval rate | 87% | > 85% | OK |
| Plat completion rate | 4/5 (80%) | > 80% | OK |
| Average time per merge | 22min | < 30min | OK |
| Escalations to patron | 1 (2%) | < 5% | OK |
| Regressions detected | 0 | 0 | OK |
| God-commis detected | 0 | 0 | OK |
| Patch Bankruptcy triggered | 1 | < 2 | OK |
| Stigmergy tokens used | 3 | - | Info |
```

| Overall score | Verdict |
|---------------|---------|
| 80-100 | Excellent — efficient brigade |
| 60-79 | Acceptable — some friction |
| 40-59 | Problematic — too many rejections/escalations |
| < 40 | Failure — replan the brigade |
