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
| `cli-forge-doc` | **Correction** — triggered by audit findings: missing CONTRIBUTING.md, architecture docs, troubleshooting |
| `cli-forge-arch` | Skip (triggered only on explicit user request) |
| `cli-forge-hld` | Skip (triggered only on explicit user request) |
| `cli-forge-lld` | Skip (triggered only on explicit user request) |
| `cli-audit-shell` | Shell scripts exist (`.sh` files or shebanged bash scripts) |
| `cli-audit-wizard` | Interactive setup/init commands exist (`setup.sh`, `init`, `doctor`, wizard UX) |
| `cli-forge-infra` | Infra files exist (`Dockerfile`, `*.tf`, `helmfile.yaml`, `k8s/`) |
| `cli-git-conventional` | **Always** — formats all commits during correction phase. Zero AI markers |
| `cli-forge-boss` | Skip (multi-agent orchestration — triggered explicitly) |

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
| `cli-audit-shell` | Shell quality score (SQI), bash anti-patterns, tooling fitness |
| `cli-audit-wizard` | Setup UX quality, config lifecycle, scriptability |

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
3. **Collect Dynamic Handoffs** from Wave 1 results (see Adaptive Handoffs below)
4. **Inject Wave 1 summaries** into Wave 2 prompts (top 3 issues + top 3 strengths per skill)
5. **Merge handoff-recommended skills** into Wave 2 (deduplicated)
6. **Launch Wave 2** — all applicable skills in parallel (planned + handoff-added)
7. **Collect Dynamic Handoffs** from Wave 2 results
8. **If Wave 2 produced new handoffs** for skills not yet run → launch **Wave 3** (adaptive, max 1 extra wave)
9. **Collect all results** for synthesis

### Adaptive Handoffs (automatic in cli-cycle context)

When cli-cycle orchestrates, handoffs are **automatic** — not just recommendations. Each skill's Dynamic Handoffs section produces structured recommendations that cli-cycle collects and acts on.

**Handoff registry format:**

```
After each wave, collect:
  {skill_name: [{caller, target_scope, reason}]}

Example after Wave 1:
  cli-audit-tangle: [
    {caller: "cli-audit-code", target: "src/core/", reason: "3 god modules detected"},
  ]
  cli-forge-pipeline: [
    {caller: "cli-audit-test", target: null, reason: "CI has no test stage"},
    {caller: "cli-audit-shell", target: ".gitlab-ci.yml", reason: "shell scripts are CI entrypoints"},
  ]
```

**Deduplication rules (tangle-aware):**

| Situation | Action |
|-----------|--------|
| Same skill + same target (or both null) | **Merge** — run once, combine reasons in the prompt |
| Same skill + different targets | **Run once per unique target** — scoped runs |
| Skill already in the current wave (planned) | **Skip** — already running |
| Skill already ran in a previous wave | **Skip** — unless the new target is a different scope |
| Skill marked "Skip" in applicability matrix | **Skip** — never auto-trigger generation skills (doc, hld, lld, arch, boss) |

**Examples:**

```
# MERGE: same skill, no specific target
cli-audit-test recommended by cli-audit-code (reason: C8 low)
cli-audit-test recommended by cli-forge-pipeline (reason: CI has no test stage)
→ Run cli-audit-test ONCE, inject both reasons into its prompt

# SCOPED: same skill, different targets
cli-audit-code recommended by cli-audit-tangle (target: src/core/, reason: god functions)
cli-audit-code recommended by cli-audit-sync (target: src/api/, reason: dead functions)
→ Run cli-audit-code TWICE: once on src/core/, once on src/api/

# SKIP: already planned
cli-forge-infra recommended by cli-audit-shell (target: deploy/)
cli-forge-infra is already in Wave 2 (planned)
→ Skip — just inject the handoff reason into cli-forge-infra's prompt

# SKIP: generation skill
cli-forge-doc recommended by cli-audit-doc (reason: missing architecture doc)
cli-forge-doc is "Skip" in the matrix
→ Skip auto-trigger — add to Recommendations section of final report instead
```

**Max waves:** 3 (Wave 1 planned + Wave 2 planned+handoffs + Wave 3 handoffs-only). Never more — if Wave 3 still produces handoffs, they go into the Recommendations section.

### Why adaptive waves work

- Wave 1 finds problems (god functions, missing tests, shell issues)
- Handoffs tell cli-cycle WHICH specific skills should investigate further and WHERE
- Wave 2 combines planned cross-cutting analysis + targeted follow-ups
- Wave 3 (rare) catches cascading discoveries
- Deduplication prevents the same skill from running 5 times because 5 different skills recommended it
- Scoped deduplication preserves value: `cli-audit-code` on `src/core/` and `src/api/` are different analyses

---

## Sub-agent prompt pattern

### Language detection (MANDATORY — before launching any sub-agent)

Before launching Wave 1, detect the project language:

```bash
# Check commit messages, README, docs, code comments
git log --oneline -10  # commit language
head -20 README.md     # doc language
```

The detected language is **injected into every sub-agent prompt**. This prevents skills from outputting English on a French project.

### Sub-agent prompt template

For each applicable skill, launch a sub-agent with:

```
Read the skill instructions from ~/.claude/skills/{skill-name}/SKILL.md
Read ~/.claude/skills/gotchas.md for known mistakes to avoid.

Then execute that skill on the project at {project-path}.

MANDATORY: Output language is {detected_language} (detected from git log and README).
Produce your report, findings, and recommendations in that exact language.
NEVER mix or switch languages within the report.

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
| Shell Quality | 6/10 | Dead fallbacks, missing getopts | cli-audit-shell |
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

### Triangulation — Convergence of Evidence

Before classifying any finding into a tier, apply **multi-method triangulation** (inspired by scientific critical thinking):

```
A finding is HIGH confidence ONLY IF detected by ≥2 independent methods.
Examples:
  - "Dead function" detected by cli-audit-tangle (call graph) AND cli-audit-code (DRY check) → HIGH
  - "Hardcoded secret" detected by cli-audit-code (C9) only → MEDIUM (single source)
  - "Missing test" detected by cli-audit-test only → MEDIUM
  - Same issue flagged by 3+ skills → HIGHEST priority in triage
```

**Deduplication rule:** Findings about the same `file:line+description` from multiple skills are MERGED into one item with `confidence = number of detecting skills`.

### GRADE-style confidence downgrading

After triangulation, each finding gets a confidence score that influences its tier placement:

| Start | Downgrade if... | Upgrade if... |
|-------|-----------------|---------------|
| HIGH (multi-method) | Single file only (-1), heuristic-based (-1) | Cross-file confirmed (+1), exact AST match (+1) |
| MEDIUM (single source) | No file:line evidence (-1), pattern-only (-1) | Multi-line context (+1) |
| LOW (heuristic) | Known false-positive pattern (-2) | Manual review confirmed (+2) |

A LOW confidence finding can still be Tier 3 if the **impact** is critical (security, data loss). Confidence = how sure we are; tier = how urgent.

### Phoenix Triage 3-2-1 — Complete correction list

**Rule: NEVER truncate. Show ALL corrections found, classified by tier.**

The triage is inspired by emergency medicine triage (treat the most critical first) and the immune system (prioritize threats by severity).

#### Tier classification criteria

| Tier | Criteria | Examples |
|------|----------|---------|
| 🔴 **Tier 3 — Critical** | Security flaws, broken functionality, data loss risk, hardcoded secrets, missing error handling causing silent failure, permissions that prevent runtime | Hardcoded passwords, `runAsUser` mismatch, missing `exit 1` on fatal error, secrets in plaintext |
| 🟡 **Tier 2 — Major** | Architecture debt, missing tests, obsolete/misleading docs, non-pinned versions, god functions, duplication with real impact, missing CI/CD | Unpinned `:latest` tags, monolithic scripts without functions, duplicated build logic, no negative tests, stale USER_STORY.md |
| 🟢 **Tier 1 — Minor** | Style, missing diagrams, nice-to-have docs, structural organization, cosmetic improvements, missing but non-critical metadata | Missing CHANGELOG/LICENSE, files in wrong directory, no Mermaid diagrams, missing Makefile |

#### Triage output template

```markdown
## 🔴 Tier 3 — Critical (N items)

| # | Correction | Effort | Source |
|---|-----------|--------|--------|
| 1 | Fix DbGate mount — /root/.dbgate but runAsUser: 1000 | Low | cli-forge-infra |
| 2 | Externalize healthcheck password — Hc!CIS2022check hardcoded in 2 places | Low | cli-audit-code |
| 3 | sed password substitution in /tmp — visible on the filesystem | Low | cli-audit-code |
| 4 | Entrypoint timeout without exit — [ $i -eq 90 ] && echo "ERROR" but no exit 1 | Low | cli-audit-code |

## 🟡 Tier 2 — Major (N items)

| # | Correction | Effort | Source |
|---|-----------|--------|--------|
| 5 | USER_STORY.md stale — describes an Ansible/Windows project that no longer exists | Medium | cli-audit-sync |
| 6 | Pin DbGate — dbgate:latest → fixed version | Low | cli-forge-infra |
| 7 | entrypoint.sh monolithic — 6 responsibilities with no functions | Medium | cli-audit-tangle |
| 8 | build.sh duplicated — identical pattern between 2 images | Medium | cli-audit-code |
| 9 | No negative tests | Medium | cli-audit-test |
| 10 | No CI/CD pipeline | High | cli-forge-pipeline |
| 11 | No K8s readiness/liveness probes | Low | cli-forge-infra |

## 🟢 Tier 1 — Minor (N items)

| # | Correction | Effort | Source |
|---|-----------|--------|--------|
| 12 | CIS XML files at the root → security/ | Low | cli-forge-tree |
| 13 | No README.md | Medium | cli-forge-readme |
| 14 | deploy/mssql.kube (quadlet) without documentation | Low | cli-audit-doc |
| 15 | No CHANGELOG, CONTRIBUTING, LICENSE | Low | cli-audit-doc |
| 16 | No Makefile/Taskfile | Medium | cli-forge-tree |
| 17 | No architecture diagram | Low | cli-forge-schema |
```

**Display rules:**
- Count per tier in the header: `🔴 Tier 3 — Critical (4 items)`
- Within each tier, sort by effort ascending (quick wins first)
- Each item has: `#` (global numbering), description, effort, source skill
- **NEVER** add a "and N more..." or "see full report" — this IS the full report

### Phoenix Choice prompt

After the triage, always present (translate into the project's detected language; this is the English template):

```markdown
---

**Phoenix — Which tier should we tackle?**

| Choice | Action | Items |
|--------|--------|-------|
| `1` | Fix everything (autonomous convergence, unified plan) | N total |
| `2` | Fix 🔴 critical only | N |
| `3` | Fix 🟡 major only | N |
| `4` | Fix 🟢 minor only | N |
| `5` | Not now | — |
```

### Correction tools (used during Phoenix fix phase)

During the fix phase (options 1-4 or convergence), cli-cycle uses **generation skills** as tools — not as audit skills. These are the "Skip" skills from the applicability matrix, activated **only** to fix triage items:

| Tool skill | When to use during correction |
|-----------|------------------------------|
| `cli-forge-doc` | Triage items about missing CONTRIBUTING.md, architecture docs, troubleshooting guide |
| `cli-forge-readme` | Triage items about missing/outdated README |
| `cli-forge-schema` | Triage items about missing diagrams |
| `cli-forge-pipeline` | Triage items about missing/broken CI stages (not just audit — can generate jobs) |
| `cli-git-conventional` | **Always** — format ALL commit messages during correction using ghostwriter style, zero AI markers |

**Workflow during correction:**
1. Fix code/config issues directly (edit files)
2. Generate missing docs via `cli-forge-doc` / `cli-forge-readme` if triage items require it
3. Generate missing diagrams via `cli-forge-schema` if triage items require it
4. Commit each logical batch using `cli-git-conventional` format (imperative, scoped, no AI trailers)
5. Re-audit

**Rule:** Generation skills are tools during correction, not standalone skills. They don't produce their full output format — they produce the specific files needed by the triage items.

### Phoenix re-audit (after fixes)

When the user chooses a tier and fixes are applied:

1. **Re-run ONLY the skills that sourced the fixed items** — not the full cycle
2. **Present a delta report:**

```markdown
## 🔥 Phoenix — Pass N

Issues resolved this pass: X
Issues remaining: Y (🔴 A / 🟡 B / 🟢 C)
New issues detected: Z (regressions from fixes)
Score: before → after
```

3. **If new issues were introduced by fixes**, add them to the triage
4. **Present the remaining triage** (updated) + the choice prompt again

### Phoenix convergence

The cycle stops when ANY of these is true:

| Condition | Output |
|-----------|--------|
| 0 items in 🔴 + 🟡 | `🔥 Phoenix — Converged. No critical or major flaws remain. N minor items left (optional).` |
| Re-audit finds 0 new issues | `🔥 Phoenix — Stable. The fixes introduced no regressions.` |
| User chooses `5` | `🔥 Phoenix — Paused. Resume with /cli-cycle.` |

### Strengths template

```markdown
## Strengths

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
