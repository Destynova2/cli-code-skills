# Skill Flow Graph — Directed Acyclic Graph of Skill Triggers

> **When to read:** During Steps 4-6 when deciding execution order, handling handoffs, and triggering forge skills from audit findings.

---

## Flow philosophy (inspired by cli-audit-tangle)

The skill ecosystem is a **directed graph**. Each edge represents a trigger condition:
- `audit-X --[condition]--> forge-Y` = audit finding triggers a forge correction
- `audit-X --[handoff]--> audit-Y` = audit recommends deeper analysis by another audit
- `forge-X --[always]--> git-conventional` = every forge output gets committed

**Anti-pattern detection:** If the graph has cycles, skills call each other forever. We use the same Tarjan SCC detection as tangle: any cycle = bug in the flow.

---

## The DAG

```
                        ┌─────────────┐
                        │  cli-cycle   │
                        │ (orchestrator)│
                        └──────┬───────┘
                               │
                 ┌─────────────┼─────────────┐
                 │             │             │
           ┌─────▼─────┐ ┌────▼────┐ ┌──────▼──────┐
           │  WAVE 1    │ │ WAVE 2  │ │  WAVE 3     │
           │ (parallel) │ │(parallel│ │ (adaptive   │
           │            │ │+handoffs│ │  handoffs)  │
           └─────┬──────┘ └────┬────┘ └──────┬──────┘
                 │             │             │
                 └─────────────┼─────────────┘
                               │
                        ┌──────▼───────┐
                        │   TRIAGE     │
                        │  3-2-1       │
                        └──────┬───────┘
                               │
                        ┌──────▼───────┐
                        │  CORRECTION  │
                        │  pipeline    │
                        └──────┬───────┘
                               │
                 ┌─────────────┼─────────────┐
                 │             │             │
           ┌─────▼─────┐ ┌────▼────┐ ┌──────▼──────┐
           │ Direct fix │ │ Forge   │ │ git-conv    │
           │ (edit code)│ │ (generate│ │ (commit)    │
           │            │ │  docs)  │ │ ALWAYS      │
           └────────────┘ └─────────┘ └─────────────┘
                               │
                        ┌──────▼───────┐
                        │  RE-AUDIT    │
                        │ (affected    │
                        │  skills only)│
                        └──────┬───────┘
                               │
                          converge?
                          ┌──▼──┐
                          │ oui │──→ FIN
                          │ non │──→ CORRECTION (boucle)
                          └─────┘
```

---

## Trigger edges (who triggers whom)

### Wave 1 → Wave 2 (Handoff edges)

Each edge fires only if the condition is detected during Wave 1.

```
cli-audit-code ──[god modules detected]──────────→ cli-audit-tangle (Wave 2)
cli-audit-code ──[C9 security issues in .sh]─────→ cli-audit-shell (already Wave 1, inject reason)
cli-audit-code ──[C8 test quality low]───────────→ cli-audit-test (already Wave 1, inject reason)
cli-audit-shell ──[sed on YAML detected]─────────→ cli-forge-infra (Wave 2)
cli-audit-shell ──[N scripts, no getopts]────────→ cli-audit-wizard (already Wave 1, inject reason)
cli-audit-shell ──[scripts are CI entrypoints]───→ cli-forge-pipeline (Wave 2)
cli-audit-tangle ──[CI deadlocks detected]───────→ cli-forge-pipeline (Wave 2)
cli-audit-tangle ──[call graph > 20 nodes]───────→ cli-forge-schema (Wave 2)
cli-audit-doc ──[docs describe dead features]────→ cli-audit-sync (Wave 2)
cli-audit-doc ──[README score low]───────────────→ cli-forge-readme (already Wave 1, inject reason)
cli-audit-test ──[no drift detection D13]────────→ cli-audit-drift (Wave 2)
cli-audit-test ──[CI has no test stage]──────────→ cli-forge-pipeline (Wave 2)
```

### Wave 2 → Wave 3 (Adaptive handoff edges)

```
cli-audit-sync ──[shell commands in docs broken]──→ cli-audit-shell (if not run in Wave 1)
cli-audit-sync ──[stale diagrams]──────────────────→ cli-forge-schema (inject reason)
cli-audit-drift ──[god-level contracted funcs]─────→ cli-audit-tangle (inject reason)
cli-forge-pipeline ──[shell entrypoints in CI]─────→ cli-audit-shell (if not run)
cli-forge-infra ──[complex deployment, no HLD]─────→ REPORT ONLY (forge-hld is Skip)
```

### Triage → Correction (Forge trigger edges)

These edges fire during the Phoenix correction phase, based on triage items.

```
Finding: "missing CONTRIBUTING.md" ────→ cli-forge-doc
Finding: "missing/outdated README" ────→ cli-forge-readme
Finding: "missing diagrams" ───────────→ cli-forge-schema
Finding: "CI missing stages" ──────────→ cli-forge-pipeline (generation mode)
Finding: "code/config issue" ──────────→ Direct edit (no forge skill needed)
ALL corrections ───────────────────────→ cli-git-conventional (ALWAYS)
```

---

## Deduplication matrix

When multiple edges point to the same skill, apply tangle-aware deduplication:

| Same skill | Same scope | Action |
|-----------|-----------|--------|
| cli-audit-tangle | null + null | **Merge** — run once, combine reasons |
| cli-audit-shell | `setup.sh` + `deploy/*.sh` | **Scoped** — run once on `deploy/` (broader scope wins) |
| cli-forge-pipeline | null + null | **Merge** |
| cli-forge-schema | null + null | **Merge** |
| cli-audit-code | `src/core/` + `src/api/` | **Scoped** — run twice, different targets |
| Any skill | Already ran in earlier wave | **Skip** — inject reason only |

**Scope resolution:** When merging scoped calls, take the **broader scope** (directory > file > function). If scopes don't overlap, run separately.

---

## Cycle detection (safety)

The flow graph MUST be acyclic. Apply Tarjan SCC check on the edge list:

**Known safe patterns:**
- `audit-X → forge-Y → git-conventional → STOP` (linear, no cycle)
- `audit-X → audit-Y → forge-Z` (forward only, no cycle)
- `re-audit → same audit skills` (this is the Phoenix LOOP, not a graph cycle — it's controlled by convergence criteria)

**Forbidden cycles:**
- `audit-X → audit-Y → audit-X` (infinite handoff ping-pong)
- `forge-X → audit-Y → forge-X` (fix creates a finding that triggers the same fix)

**Prevention rules:**
1. A handoff edge can only point **forward** (Wave 1 → Wave 2 → Wave 3, never backward)
2. A skill can recommend itself ONLY with a **different scope** (not the same file)
3. Wave 3 is terminal — no Wave 4. Remaining handoffs go to the Recommendations section
4. During correction, if a fix triggers a new finding that would re-trigger the same forge skill → STOP, flag as divergence

---

## Execution order summary

```
Phase 1: AUDIT
  Wave 1 (parallel): code, doc, test, tangle, tree, readme, shell, wizard
    ↓ collect handoffs
  Wave 2 (parallel): sync, drift, pipeline, schema, infra + handoff-added skills
    ↓ collect handoffs
  Wave 3 (if needed): handoff-only skills not yet run
    ↓
  Triage 3-2-1

Phase 2: CORRECTION (for each triage item)
  Fix item → forge if needed → git-conventional commit
    ↓
  Re-audit (affected skills only)
    ↓
  Converge? → YES: done | NO: next item or next pass

Phase 3: REPORT
  Score card + triage + strengths + trends + convergence status
```

---

## Forge skill modes

Each forge skill operates in 2 modes depending on context:

| Mode | Triggered by | Output |
|------|-------------|--------|
| **Audit mode** (Wave 1-3) | Applicability matrix condition | Score + findings (read-only) |
| **Correction mode** (Phoenix fix) | Triage item requiring generation | Generated files (write) |

Skills that operate in both modes:
- `cli-forge-readme` — audit: scores README (RCI). correction: rewrites README
- `cli-forge-tree` — audit: scores structure (SHS). correction: reorganizes files
- `cli-forge-pipeline` — audit: scores pipeline (PSI). correction: adds/fixes CI jobs
- `cli-forge-schema` — audit: checks existing diagrams. correction: generates new diagrams
- `cli-forge-infra` — audit: scores infra (IDS/DHS). correction: simplifies config

Skills that operate in correction mode only:
- `cli-forge-doc` — generates CONTRIBUTING.md, architecture docs, troubleshooting
- `cli-git-conventional` — formats all commits (always active during correction)
