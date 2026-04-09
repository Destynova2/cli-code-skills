# Phoenix Autonomous Convergence — Dry-Run Multi-Pass

> **When to read:** When the user chooses option `[6]` in the Phoenix Choice, or invokes `/cli-cycle --converge [target-score]`.

---

## Concept

Instead of the interactive loop (audit → user chooses → fix → re-audit → repeat), Autonomous Convergence mode runs the entire correction cycle **autonomously in an isolated worktree**, accumulating ALL findings and fixes across N passes, then presents a **single unified plan** with the complete diff.

```
Normal mode:                          Autonomous Convergence:

  Audit → Show → User picks            Audit ──┐
    ↓                                    Fix    │ Worktree
  Fix tier → Re-audit → Show            ↓      │ (isolated)
    ↓                                  Re-audit │
  User picks again                       Fix    │
    ↓                                    ↓      │
  ...N interactions...                 Converge ─┘
                                         ↓
                                       Show unified plan
                                         ↓
                                       User approves/rejects
                                         ↓
                                       Apply ALL or nothing
```

**Key advantage:** The user reviews ONE comprehensive plan instead of making N decisions. The plan includes corrections that the first audit couldn't find because they only appear after earlier fixes are applied.

---

## Workflow

### Step C1 — Create isolated worktree

```bash
git worktree add /tmp/phoenix-convergence-$(date +%s) -b phoenix/convergence HEAD
```

All subsequent work happens in the worktree. The user's working directory is never touched until they approve.

### Step C2 — Initial full audit (Pass 0)

Run the normal cli-cycle Steps 1-5 (discover skills, detect context, run waves, collect findings). This is Pass 0 — the baseline.

Record:
- `pass_0_score`: the initial overall score
- `pass_0_findings[]`: all findings with tier, description, file, source skill

### Step C3 — Autonomous fix loop

**File-based progress tracking (MANDATORY — prevents the LLM from "forgetting" items):**

Before starting the loop, write the full triage to a checkpoint file:

```bash
mkdir -p .claude/cycle-progress
cat > .claude/cycle-progress/current.md <<'EOF'
# Cycle Progress — Pass {N}

## Tier 3 (N items)
- [ ] #1 description (skill: cli-audit-X)
- [ ] #2 description (skill: cli-audit-Y)
...

## Tier 2 (M items)
- [ ] #N+1 description
...

## Tier 1 effort=Low (K items)
- [ ] #N+M+1 description
...
EOF
```

Then iterate **explicitly** with checkbox updates:

```
pass = 1
target_score = user_specified OR default 8.5
max_passes = user_specified OR default 10
accumulated_findings = pass_0_findings

WHILE (has_tier3 OR has_tier2) AND pass <= max_passes:

  1. FOR EACH item in tier3 (ordered):
       a. Read the item from .claude/cycle-progress/current.md
       b. Apply the fix (direct edit OR forge skill OR cli-git-conventional)
       c. Verify the fix succeeded (file changed, test passes)
       d. Mark the item as [x] in current.md (sed -i "s/- \[ \] #${N}/- [x] #${N}/")
       e. Append a log line to .claude/cycle-progress/log.txt
       NEVER skip an item. NEVER batch-mark multiple items as done.
       If a fix fails: mark as [!] (FAILED) and continue, don't stop.

  2. FOR EACH item in tier2 (same protocol)

  3. FOR EACH item in tier1 IF effort == "Low" (same protocol)

  4. Verify all items are [x] or [!] in current.md before continuing
     If any [ ] remains → ERROR, you skipped items, restart the loop

  5. Re-run ONLY the skills that sourced the fixed items
  6. Collect new findings (regressions + newly visible issues)
  7. Append new findings as [ ] to current.md (cascades)
  8. Compute new score
  9. EARLY-STOP checks (any true = BREAK):
     a. score >= target_score AND no Tier 3         → CONVERGED
     b. new_findings_count == 0                      → STABLE (no regressions)
     c. new_tier3_count > resolved_tier3_count       → DIVERGING (stop, report)
     d. score_delta < 0.1 for 2 consecutive passes   → PLATEAU (diminishing returns)
  10. pass++
```

### Step C4 — Deduplicate and classify

After convergence, the accumulated findings list contains:
- **Resolved items**: found in pass N, fixed in pass N, confirmed resolved in pass N+1
- **Cascaded items**: NOT found in pass 0, appeared only after earlier fixes were applied
- **Persistent items**: found in pass 0, still present (typically Tier 1 "skip" items like CHANGELOG/LICENSE)

Classify each item:

```
| Status   | Meaning | In plan? |
|----------|---------|----------|
| RESOLVED | Fixed during convergence | YES (included in diff) |
| CASCADED | Only appeared after earlier fix | YES (included in diff, marked as cascade) |
| SKIPPED  | Tier 1 with Effort > Low, or decision-level items | NO (listed as remaining) |
| DEFERRED | Items that require user decision (architecture, licensing) | NO (listed for user) |
```

### Step C5 — Build unified diff

```bash
# In the worktree, after all passes:
git diff HEAD~0..HEAD  # if committed per pass
# OR
git diff main..phoenix/convergence  # total diff from original
```

### Step C6 — Present unified plan to user

The output has 4 sections (translate into the project's detected language; this is the English template):

```markdown
# Phoenix Autonomous Convergence -- {project-name}

**Initial score**: X.X/10
**Final score**: Y.Y/10
**Passes performed**: N
**Files modified**: M
**Items resolved**: A / B total (C cascades detected)

## Pass timeline

| Pass | Score | Tier 3 | Tier 2 | Tier 1 | Items resolved | Cascades |
|------|-------|--------|--------|--------|----------------|----------|
| 0 (baseline) | 6.8 | 4 | 7 | 6 | -- | -- |
| 1 | 7.5 | 0 | 4 | 5 | 8 | 2 new |
| 2 | 8.3 | 0 | 1 | 4 | 5 | 1 new |
| 3 | 8.7 | 0 | 0 | 3 | 2 | 0 |
| **Converged** | **8.7** | **0** | **0** | **3** | **15 total** | **3 total** |

## Applied corrections (A items)

### Pass 1 (direct corrections)
| # | Correction | Tier | Source | Files |
|---|-----------|------|--------|-------|
| 1 | HEALTHCHECK_PASSWORD externalized | Tier 3 | cli-audit-code | Containerfile, entrypoint.sh |
| 2 | sed temp file -> direct pipe | Tier 3 | cli-audit-code | entrypoint.sh |
| ... |

### Pass 2 (cascades = issues surfaced after pass 1)
| # | Correction | Tier | Source | Cause | Files |
|---|-----------|------|--------|-------|-------|
| 9 | HEALTHCHECK_PASSWORD missing in Makefile test | Tier 2 | cli-audit-sync | Cascade of #1 | Makefile |
| 10 | HEALTHCHECK CMD exposes password via podman inspect | Tier 3 | cli-audit-code | Cascade of #1 | Containerfile |
| ... |

### Pass 3 (stabilization)
| ... |

## Remaining items (not fixed)

| # | Item | Tier | Reason |
|---|------|------|--------|
| 16 | No CHANGELOG.md | Tier 1 | Business decision |
| 17 | No LICENSE | Tier 1 | Business decision |
| 18 | Missing prod overlay | Tier 2 | Architecture decision |

## Full diff

[The complete git diff is available on branch phoenix/convergence]
Modified files: {list}

---

**Phoenix -- Apply the plan?**

| Choice | Action |
|--------|--------|
| `yes` | Merge the phoenix/convergence branch into the current branch |
| `partial` | Show the diff file-by-file for cherry-picking |
| `no` | Delete the branch and worktree, no modifications |
```

### Step C7 — Apply or discard

| User choice | Action |
|-------------|--------|
| `yes` | `git merge phoenix/convergence` in the main working directory |
| `partial` | Show diff per file, user cherry-picks |
| `no` | `git worktree remove /tmp/phoenix-*; git branch -D phoenix/convergence` |

---

## Convergence criteria

The loop stops when ANY of these is true:

The primary stop mechanism is **early-stop** (Step C3.9), not the max pass count. The max is a safety net, not the expected exit.

| Condition | Name | Meaning |
|-----------|------|---------|
| 0 Tier 3 AND 0 Tier 2 | **CONVERGED** | Healthy — only cosmetic items remain |
| Score >= target_score | **CONVERGED** | User's target reached |
| Re-audit finds 0 new issues | **STABLE** | No regressions — fixes are clean |
| Score delta < 0.1 for 2 consecutive passes | **PLATEAU** | Diminishing returns — remaining items are hard/deferred |
| A fix introduces MORE Tier 3 than it resolves | **DIVERGING** | Stop immediately — corrections are making things worse |
| Pass > max_passes (default 10) | **SAFETY** | Unlikely to reach — early-stop should trigger first |

---

## Safety rules

1. **Never modify the user's working directory** until they explicitly approve
2. **Commit after each pass** in the worktree with message `phoenix: pass N — X items resolved`
3. **Stop if diverging**: if a pass introduces more Tier 3 items than it resolves, stop and present what you have
4. **Max passes is a safety net, not the exit condition**: early-stop (converged/stable/plateau/diverging) is the primary mechanism. Default max is 10 but configurable via `--max-passes`. In practice, convergence happens in 2-4 passes
5. **No architecture decisions**: items marked as "business decision" or "architecture decision" are DEFERRED, never auto-fixed
6. **Track cascades explicitly**: when a fix in pass N causes a new finding in pass N+1, record the causal link

---

## Cascade detection

A cascade is a finding that:
1. Did NOT exist in any previous pass
2. Appeared in pass N+1
3. Is causally linked to a fix applied in pass N

Detection heuristic:
```
For each new finding in pass N+1:
  For each fix applied in pass N:
    IF new_finding.file overlaps with fix.files
    OR new_finding.description references same concept (e.g., same variable name, same config key)
    THEN mark as cascade of that fix
```

Cascades are **expected and valuable** — they're the whole reason this mode exists. The user sees them explicitly labeled, which builds trust in the process.

---

## Deferred items — What NOT to auto-fix

| Category | Examples | Why |
|----------|---------|-----|
| Licensing | CHANGELOG, LICENSE, CONTRIBUTING | Business/legal decision |
| Architecture | Overlay prod, module split, language rewrite | Design decision |
| External deps | Pin Trivy/Grype to specific version | Team SSI manages this |
| Conventions | Rename variables across multi-team project | Coordination needed |
| Features | Add startupProbe, add --dry-run flag | Scope decision |

These items appear in the "Remaining items" section with their reason.

---

## Invocation

```bash
# Default: converge until 0 Tier 3 + 0 Tier 2
/cli-cycle --converge

# With target score
/cli-cycle --converge 9

# With max passes override
/cli-cycle --converge 8.5 --max-passes 3
```

Or via the Phoenix Choice menu:
```
[6] Autonomous convergence (dry-run, fix everything, propose a unified plan)
```
