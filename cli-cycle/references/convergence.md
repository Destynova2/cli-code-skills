# Phoenix Convergence Autonome — Dry-Run Multi-Pass

> **When to read:** When the user chooses option `[6]` in the Phoenix Choice, or invokes `/cli-cycle --converge [target-score]`.

---

## Concept

Instead of the interactive loop (audit → user chooses → fix → re-audit → repeat), the Convergence Autonome mode runs the entire correction cycle **autonomously in an isolated worktree**, accumulating ALL findings and fixes across N passes, then presents a **single unified plan** with the complete diff.

```
Normal mode:                          Convergence Autonome:

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

```
pass = 1
target_score = user_specified OR default 8.5
max_passes = 5
accumulated_findings = pass_0_findings

WHILE (has_tier3 OR has_tier2) AND pass <= max_passes:
  1. Fix ALL Tier 3 items from current findings
  2. Fix ALL Tier 2 items from current findings
  3. Fix Tier 1 items IF effort = "Faible" only
  4. Re-run ONLY the skills that sourced the fixed items
  5. Collect new findings (regressions + newly visible issues)
  6. Add new findings to accumulated_findings (deduplicate by file:line + description similarity)
  7. Mark resolved findings as RESOLVED
  8. Compute new score
  9. IF score >= target_score AND no Tier 3: BREAK (converged)
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
| SKIPPED  | Tier 1 with Effort > Faible, or decision-level items | NO (listed as remaining) |
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

The output has 4 sections:

```markdown
# Phoenix Convergence Autonome -- {project-name}

**Score initial**: X.X/10
**Score final**: Y.Y/10
**Passes effectuees**: N
**Fichiers modifies**: M
**Items resolus**: A / B total (C cascades detectees)

## Chronologie des passes

| Passe | Score | Tier 3 | Tier 2 | Tier 1 | Items resolus | Cascades |
|-------|-------|--------|--------|--------|---------------|----------|
| 0 (baseline) | 6.8 | 4 | 7 | 6 | -- | -- |
| 1 | 7.5 | 0 | 4 | 5 | 8 | 2 nouveaux |
| 2 | 8.3 | 0 | 1 | 4 | 5 | 1 nouveau |
| 3 | 8.7 | 0 | 0 | 3 | 2 | 0 |
| **Convergence** | **8.7** | **0** | **0** | **3** | **15 total** | **3 total** |

## Corrections appliquees (A items)

### Passe 1 (corrections directes)
| # | Correction | Tier | Source | Fichiers |
|---|-----------|------|--------|----------|
| 1 | HEALTHCHECK_PASSWORD externalise | Tier 3 | cli-audit-code | Containerfile, entrypoint.sh |
| 2 | sed temp file -> pipe direct | Tier 3 | cli-audit-code | entrypoint.sh |
| ... |

### Passe 2 (cascades = issues apparues apres passe 1)
| # | Correction | Tier | Source | Cause | Fichiers |
|---|-----------|------|--------|-------|----------|
| 9 | HEALTHCHECK_PASSWORD manquant dans Makefile test | Tier 2 | cli-audit-sync | Cascade de #1 | Makefile |
| 10 | HEALTHCHECK CMD expose le mdp via podman inspect | Tier 3 | cli-audit-code | Cascade de #1 | Containerfile |
| ... |

### Passe 3 (stabilisation)
| ... |

## Items restants (non corriges)

| # | Item | Tier | Raison |
|---|------|------|--------|
| 16 | Pas de CHANGELOG.md | Tier 1 | Decision metier |
| 17 | Pas de LICENSE | Tier 1 | Decision metier |
| 18 | Overlay prod manquant | Tier 2 | Decision architecture |

## Diff complet

[Le diff git complet est disponible dans la branche phoenix/convergence]
Fichiers modifies: {list}

---

**Phoenix -- Appliquer le plan ?**

| Choix | Action |
|-------|--------|
| `oui` | Merge la branche phoenix/convergence dans la branche courante |
| `partiel` | Afficher le diff fichier par fichier pour selection |
| `non` | Supprimer la branche et le worktree, aucune modification |
```

### Step C7 — Apply or discard

| User choice | Action |
|-------------|--------|
| `oui` | `git merge phoenix/convergence` in the main working directory |
| `partiel` | Show diff per file, user cherry-picks |
| `non` | `git worktree remove /tmp/phoenix-*; git branch -D phoenix/convergence` |

---

## Convergence criteria

The loop stops when ANY of these is true:

| Condition | Meaning |
|-----------|---------|
| 0 Tier 3 AND 0 Tier 2 | Healthy — only cosmetic items remain |
| Score >= target_score | User's target reached |
| Pass > max_passes (5) | Safety limit — prevent infinite loops |
| Re-audit finds 0 new issues | Stable — no regressions from fixes |
| A fix introduces MORE Tier 3 than it resolves | Diverging — stop and report |

---

## Safety rules

1. **Never modify the user's working directory** until they explicitly approve
2. **Commit after each pass** in the worktree with message `phoenix: pass N — X items resolved`
3. **Stop if diverging**: if a pass introduces more Tier 3 items than it resolves, stop and present what you have
4. **Max 5 passes**: prevent runaway loops. If not converged after 5, present current state
5. **No architecture decisions**: items marked as "decision metier" or "decision architecture" are DEFERRED, never auto-fixed
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

These items appear in the "Items restants" section with their reason.

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
[6] Convergence autonome (dry-run, corrige tout, propose un plan unifie)
```
