# Orchestration — Skill Applicability & Execution

> **When to read:** During Steps 3-5 when deciding which skills to run, launching sub-agents, and synthesizing results.

---

## Skill applicability matrix

| Skill | Run if... |
|-------|----------|
| `cli-audit-code` | Source code files exist |
| `cli-audit-doc` | Documentation files exist (`.md`, doc comments) |
| `cli-audit-sync` | Both source code AND documentation exist (verifies coherence between them) |
| `cli-audit-test` | Test files or test directory exists |
| `cli-audit-drift` | CONTRACTS.md exists (or bootstrap if critical functions detected and no contracts) |
| `cli-audit-tangle` | Source code files exist (analyzes call graph topology) |
| `cli-forge-readme` | `README.md` exists (audit mode: compare current vs ideal) |
| `cli-forge-tree` | Always (audit current structure) |
| `cli-forge-schema` | Existing Mermaid diagrams found OR architecture worth diagramming |
| `cli-forge-pipeline` | CI config exists (`.github/workflows/`, `.gitlab-ci.yml`, `Jenkinsfile`) |
| `cli-forge-doc` | Skip (generation, not audit — unless no docs exist at all) |
| `cli-forge-arch` | Skip (generation, not audit — unless user explicitly asks) |
| `cli-forge-hld` | Skip (generation — use only when user asks to design/architect) |
| `cli-forge-lld` | Skip (generation — use only when user asks for detailed design) |
| `cli-forge-infra` | Infra files exist (`Dockerfile`, `*.tf`, `helmfile.yaml`, `k8s/`) |

---

## Execution order — 2 waves

Skills have dependencies. Run in 2 waves, not all at once.

### Wave 1 — Foundation (parallel)

These skills scan the codebase independently. Launch all in parallel.

| Skill | What it produces |
|-------|-----------------|
| `cli-audit-code` | Code quality score, anti-patterns, module structure |
| `cli-audit-doc` | Doc quality score, anti-patterns, coverage |
| `cli-audit-test` | Test quality score, techniques, pyramid shape, D13 drift detection |
| `cli-audit-tangle` | Call graph topology, god functions, cycles, module coupling |
| `cli-forge-tree` | Project structure audit |
| `cli-forge-readme` | README completeness audit (RCI) |

### Wave 2 — Cross-cutting (parallel, after Wave 1)

These skills benefit from Wave 1 results or cross-reference multiple concerns.

| Skill | Why it needs Wave 1 | What it uses |
|-------|-------------------|-------------|
| `cli-audit-sync` | Compares docs against code — needs to know what code and docs exist | Code index from audit-code context, doc inventory from audit-doc |
| `cli-audit-drift` | Scans code against contracts — benefits from knowing the code structure | Code structure awareness |
| `cli-forge-pipeline` | Audits CI — benefits from knowing test structure and quality | Test pyramid shape from audit-test, test techniques detected |
| `cli-forge-schema` | Audits diagrams against code — needs to know module structure | Module structure from audit-code |
| `cli-forge-infra` | Audits infra files — independent but lower priority | — |

### Wave execution rules

1. **Launch Wave 1** — all applicable skills in parallel
2. **Wait for Wave 1 to complete**
3. **Inject Wave 1 summaries** into Wave 2 prompts (top 3 issues + top 3 strengths per skill)
4. **Launch Wave 2** — all applicable skills in parallel
5. **Collect all results** for synthesis

### Why 2 waves instead of all parallel?

- Wave 2 skills that cross-reference (sync, drift, pipeline) produce **better results** when they know the code/doc/test landscape
- Wave 1 skills are independent — no benefit from ordering them
- 2 waves is the right granularity: finer (3+ waves) adds latency without quality gain

---

## Sub-agent prompt pattern

For each applicable skill, launch a sub-agent with:

```
Read the skill instructions from ~/.claude/skills/{skill-name}/SKILL.md

Then execute that skill on the project at {project-path}.

Important:
- Follow the skill's workflow exactly as written
- Read any reference files the skill mentions (references/*.md, reference.md)
- Output a structured result with:
  1. Overall score (if the skill produces one)
  2. Top 3 critical issues found
  3. Top 3 things done well
  4. Full detailed report

Target: {project-path}
Arguments: {appropriate args for this skill}
```

For **Wave 2** skills, append Wave 1 context to the prompt:

```
Context from prior audits (use to enrich your analysis, do not re-scan):
- Code quality: {score}/10 — top issues: {list}
- Doc quality: {score}/10 — top issues: {list}
- Test quality: {score}/100 — pyramid: {shape}, techniques: {list}
- Structure: {summary}
```

---

## Synthesis — Report templates

### Score card

```markdown
## Project Health — {project-name}

| Area | Score | Status | Skill |
|------|-------|--------|-------|
| Code Quality | 7/10 | Needs work | cli-audit-code |
| Doc-Code Sync | 6/10 | Drift detected | cli-audit-sync |
| Documentation | 8/10 | Good | cli-audit-doc |
| Test Quality | 5/10 | Gaps | cli-audit-test |
| Code Topology | 7/10 | 2 god functions | cli-audit-tangle |
| Semantic Drift | N/A | No CONTRACTS.md | cli-audit-drift |
| README | 6/10 | Outdated | cli-forge-readme |
| Project Structure | 9/10 | Excellent | cli-forge-tree |
| Diagrams | 4/10 | Missing | cli-forge-schema |
| Pipeline | 7/10 | Good | cli-forge-pipeline |
| Infrastructure | N/A | No infra files | cli-forge-infra |
| **Overall** | **6.8/10** | | |
```

### Phoenix Triage 3-2-1 — Complete correction list

**Rule: NEVER truncate. Show ALL corrections found, classified by tier.**

The triage is inspired by emergency medicine triage (treat the most critical first) and the immune system (prioritize threats by severity).

#### Tier classification criteria

| Tier | Criteria | Examples |
|------|----------|---------|
| 🔴 **Tier 3 — Critique** | Security flaws, broken functionality, data loss risk, hardcoded secrets, missing error handling causing silent failure, permissions that prevent runtime | Hardcoded passwords, `runAsUser` mismatch, missing `exit 1` on fatal error, secrets in plaintext |
| 🟡 **Tier 2 — Majeur** | Architecture debt, missing tests, obsolete/misleading docs, non-pinned versions, god functions, duplication with real impact, missing CI/CD | Unpinned `:latest` tags, monolithic scripts without functions, duplicated build logic, no negative tests, stale USER_STORY.md |
| 🟢 **Tier 1 — Mineur** | Style, missing diagrams, nice-to-have docs, structural organization, cosmetic improvements, missing but non-critical metadata | Missing CHANGELOG/LICENSE, files in wrong directory, no Mermaid diagrams, missing Makefile |

#### Triage output template

```markdown
## 🔴 Tier 3 — Critique (N items)

| # | Correction | Effort | Source |
|---|-----------|--------|--------|
| 1 | Fix DbGate mount — /root/.dbgate mais runAsUser: 1000 | Faible | cli-forge-infra |
| 2 | Externaliser le mdp healthcheck — Hc!CIS2022check hardcodé en 2 endroits | Faible | cli-audit-code |
| 3 | Substitution sed de mdp dans /tmp — visible sur le filesystem | Faible | cli-audit-code |
| 4 | Timeout entrypoint sans exit — [ $i -eq 90 ] && echo "ERREUR" mais pas de exit 1 | Faible | cli-audit-code |

## 🟡 Tier 2 — Majeur (N items)

| # | Correction | Effort | Source |
|---|-----------|--------|--------|
| 5 | USER_STORY.md obsolète — décrit un projet Ansible/Windows qui n'existe plus | Moyen | cli-audit-sync |
| 6 | Pinner DbGate — dbgate:latest → version fixe | Faible | cli-forge-infra |
| 7 | entrypoint.sh monolithique — 6 responsabilités sans fonctions | Moyen | cli-audit-tangle |
| 8 | build.sh dupliqué — pattern identique entre 2 images | Moyen | cli-audit-code |
| 9 | Pas de tests négatifs | Moyen | cli-audit-test |
| 10 | Pas de CI/CD pipeline | Fort | cli-forge-pipeline |
| 11 | Pas de readiness/liveness probes K8s | Faible | cli-forge-infra |

## 🟢 Tier 1 — Mineur (N items)

| # | Correction | Effort | Source |
|---|-----------|--------|--------|
| 12 | Fichiers XML CIS à la racine → security/ | Faible | cli-forge-tree |
| 13 | Pas de README.md | Moyen | cli-forge-readme |
| 14 | deploy/mssql.kube (quadlet) sans documentation | Faible | cli-audit-doc |
| 15 | Pas de CHANGELOG, CONTRIBUTING, LICENSE | Faible | cli-audit-doc |
| 16 | Pas de Makefile/Taskfile | Moyen | cli-forge-tree |
| 17 | Pas de diagramme d'architecture | Faible | cli-forge-schema |
```

**Display rules:**
- Count per tier in the header: `🔴 Tier 3 — Critique (4 items)`
- Within each tier, sort by effort ascending (quick wins first)
- Each item has: `#` (global numbering), description, effort, source skill
- **NEVER** add a "and N more..." or "see full report" — this IS the full report

### Phoenix Choice prompt

After the triage, always present:

```markdown
---

**Phoenix — Quel tier attaquer ?**

| Choix | Action | Items |
|-------|--------|-------|
| `1` | Tout corriger (🔴 → 🟡 → 🟢) | N total |
| `2` | Corriger les 🔴 critiques | N |
| `3` | Corriger les 🟡 majeurs | N |
| `4` | Corriger les 🟢 mineurs | N |
| `5` | Pas maintenant | — |
```

### Phoenix re-audit (after fixes)

When the user chooses a tier and fixes are applied:

1. **Re-run ONLY the skills that sourced the fixed items** — not the full cycle
2. **Present a delta report:**

```markdown
## 🔥 Phoenix — Passe N

Issues résolues cette passe : X
Issues restantes : Y (🔴 A / 🟡 B / 🟢 C)
Nouvelles issues détectées : Z (regressions from fixes)
Score : avant → après
```

3. **If new issues were introduced by fixes**, add them to the triage
4. **Present the remaining triage** (updated) + the choice prompt again

### Phoenix convergence

The cycle stops when ANY of these is true:

| Condition | Output |
|-----------|--------|
| 0 items in 🔴 + 🟡 | `🔥 Phoenix — Convergé. Aucun défaut critique ou majeur. N mineurs restants (optionnels).` |
| Re-audit finds 0 new issues | `🔥 Phoenix — Stable. Les corrections n'ont pas introduit de régressions.` |
| User chooses `5` | `🔥 Phoenix — Pausé. Reprendre avec /cli-cycle.` |

### Strengths template

```markdown
## Points forts

- Clean module boundaries (cli-audit-code: 9/10 on Structure)
- Excellent naming conventions (cli-audit-code: 9/10 on Naming)
- README has working quickstart (cli-forge-readme: verified)
```

### Trends (if previous audit exists)

```markdown
## Trends

| Area | Previous | Current | Delta |
|------|----------|---------|-------|
| Code Quality | 6/10 | 7/10 | +1 |
| Documentation | 8/10 | 8/10 | = |
| README | 4/10 | 6/10 | +2 |
```
