# Brigade Anti-Patterns — Detection and Resolution

> **When to read:** During execution (Phase 2+), to identify and fix brigade dysfunctions.

---

## Named anti-patterns

| # | Name | Detection | Severity | Resolution |
|---|------|-----------|----------|------------|
| B1 | **Ping-Pong Rejection** | Same commis × same sous-chef → rejects the same diff 3+ times | Critical | Patch Bankruptcy: reassign or redefine the spec |
| B2 | **Hot File Thrashing** | 3+ commis edit the same file during the same sprint | Major | Sequence the merges (G15); mark the file as a sensitive zone (3/3) |
| B3 | **Ghost Commis** | Commis assigned but no commit/message for > 30min | Major | Timeout, reassign, diagnose (NFD: can it communicate?) |
| B4 | **Gate Consensus Flip** | Sous-chef approves a diff then rejects it in the next round (or vice versa) | Major | Audit the sous-chef — its rules are inconsistent |
| B5 | **God Commis** | Commis touches > 50% of sprint files OR > 5 independent tasks | Major | Split, reassign, add another commis |
| B6 | **Speculative Task** | Commis starts before its dependency tokens are dropped | Minor | Remind the stigmergy rule; if intentional, mark as EXPLORATION |
| B7 | **Dead Branch** | A commis's branch has existed > 2 sprints without merge | Minor | Archive, clean up, notify |
| B8 | **Inter-Commis Feature Envy** | Commis A depends on > 3 outputs from Commis B | Major | Co-assign to the same commis OR precompute the shared dependency |
| B9 | **Sous-Chef Bypass** | Commis merges directly without going through the vote | Critical | Block the permissions; a commis must NEVER push to the main branch |
| B10 | **Over-Staffing** | 5+ commis for < 3 independent tasks | Minor | Shrink the brigade (mitosis tier S/M) |

---

## Detection heuristics

### B1 — Ping-Pong Rejection
```
For each (commis, sous-chef) pair:
  count = number of consecutive rejections on the same file/diff
  IF count >= 3 → PATCH BANKRUPTCY
```

### B3 — Ghost Commis
```
For each active commis:
  last_activity = max(last commit, last SendMessage, last shared-state edit)
  IF now - last_activity > 30min → GHOST
  NFD diagnosis:
    1. Can the commis read shared-state.md? (read test)
    2. Can the commis write to its worktree? (write test)
    3. Can the commis send a SendMessage? (comm test)
    IF any fails → communication problem, not performance
```

### B5 — God Commis
```
For each commis:
  files_ratio = files_touched(commis) / total_files_sprint
  task_count = number of assigned tasks
  IF files_ratio > 0.5 OR task_count > 5 → GOD COMMIS
```

### B8 — Inter-Commis Feature Envy
```
For each (commis_A, commis_B) pair:
  deps = number of tokens from A consumed by B
  IF deps > 3 → FEATURE ENVY
  → Co-assign or precompute
```
