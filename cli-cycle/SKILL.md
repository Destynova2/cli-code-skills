---
name: cli-cycle
description: "Continuous improvement cycle — orchestrates all cli-* skills on the current project, synthesizes results, and proposes prioritized improvements. Use when the user wants a full project review, a health check, a weekly cycle, or says 'audit everything', 'review the project', 'health check', 'what should I improve', 'run all audits', 'cycle', 'improvement cycle'. Designed for recurring use with '/loop 7d /cli-cycle'."
argument-hint: "[project-path]"
context: fork
agent: general-purpose
allowed-tools:
  - Read
  - Write
  - Grep
  - Glob
  - Bash
  - Agent
---

> **Optimization:** This skill uses on-demand loading. Heavy content lives in `references/` and is loaded only when needed.

> **Language rule:** Detect the project's primary language (from README, comments, docs, commit messages). Output your report in that language. If the project is bilingual, ask the user which language to use before proceeding.

# Cycle — Phoenix Continuous Improvement

> *"Le phénix ne meurt pas — il brûle ce qui est impur et renaît plus fort.
> Chaque cycle consume les défauts. Quand il n'y a plus rien à brûler, le projet est né."*

Run every applicable `cli-*` skill on the current project, collect results, deliver the **complete** list of corrections in 3 tiers, and loop until clean.

## Core Principles

1. **You don't duplicate logic — you delegate.** Each sub-agent reads the real SKILL.md and follows its instructions. You orchestrate, collect, and judge.
2. **Show everything.** Never truncate the correction list. The user sees ALL issues, organized by severity.
3. **Phoenix loop.** Audit → user fixes → re-audit → repeat until convergence.
4. **Convergence autonome.** When invoked with `--converge`, run the entire loop in an isolated worktree, then present a single unified plan. Read `references/convergence.md` for the full algorithm.
5. **Flow graph.** Skills form a DAG: audit detects → forge corrects → git commits → re-audit verifies. No cycles allowed. Read `references/skill-flow.md` for the complete trigger graph, deduplication rules, and cycle detection.

**Gotchas** — read `../../gotchas.md` before producing output to avoid known mistakes.

## Workflow

### Step 1 — Discover available skills

```
Glob: ~/.claude/skills/cli-*/SKILL.md
```

Parse each SKILL.md frontmatter to extract: `name`, `description`, `argument-hint`.

### Step 2 — Detect project context

1. Read manifest (`Cargo.toml`, `package.json`, `go.mod`, etc.)
2. Check for docs: `README.md`, `docs/`, `*.md`
3. Check for schemas/diagrams: Mermaid blocks in `.md` files
4. Check for infra: `Dockerfile`, `docker-compose.yml`, `*.tf`, `helmfile.yaml`
5. Check for tests: `tests/`, `test/`, `spec/`, `e2e/`
6. Check for contracts: `CONTRACTS.md`
7. Check for CI: `.github/workflows/`, `.gitlab-ci.yml`
8. Get project size: file count, primary language(s)

### Step 3 — Decide which skills to run

Read `references/orchestration.md` for the full applicability matrix. Key rule: **don't run skills that don't apply** — no infra files = no infra audit.

### Step 4 — Execute in 2 waves

Read `references/orchestration.md` for execution order, prompt patterns, and wave rules.

**Wave 1 (foundation, parallel):** code, doc, test, tangle, tree, readme, shell, wizard — independent scans
**Wave 2 (cross-cutting + handoffs, parallel):** sync, drift, pipeline, schema, infra + skills recommended by Wave 1 handoffs
**Wave 3 (adaptive, if needed):** skills recommended by Wave 2 handoffs that haven't run yet

Wave 2 skills get Wave 1 summaries injected. Handoff-recommended skills get the caller's reason injected. Deduplication prevents duplicate runs (same skill + same scope = run once). See `references/orchestration.md` for the full adaptive handoff algorithm.

### Step 5 — Collect, classify, and present ALL corrections

Read `references/orchestration.md` for templates and the full Phoenix triage rules.

**Critical rule: show every correction found. Never truncate.**

Produce a unified report with:

1. **Score card** — all skill scores in one table
2. **Triage 3-2-1** — ALL corrections, classified in 3 tiers (see below)
3. **Strengths** — what the project does right
4. **Trends** — if previous audit exists in `docs/reviews/`, compare scores
5. **Phoenix status** — convergence indicator

#### Triage 3-2-1 (inspired by emergency medicine triage + immune system)

Every correction goes into exactly one tier. The tiers define **treatment order**, not importance:

| Tier | Name | Criteria | Action |
|------|------|----------|--------|
| **🔴 Tier 3** | **Critique** | Security flaws, broken functionality, data loss risk, hardcoded secrets, missing error handling that causes silent failure | Fix NOW — these block everything |
| **🟡 Tier 2** | **Majeur** | Architecture debt, missing tests, obsolete docs, non-pinned versions, code smells with real impact (god functions, duplication) | Fix NEXT — these degrade over time |
| **🟢 Tier 1** | **Mineur** | Style, missing diagrams, nice-to-have docs, structural organization, cosmetic improvements | Fix LATER — these improve but don't protect |

**Display rules:**
- Show ALL items in each tier — no "top N" truncation
- Each item gets: `#`, description, effort (Faible/Moyen/Fort), source skill
- Count per tier shown in the tier header: `🔴 Tier 3 — Critique (4 items)`

### Step 6 — Phoenix Choice

After presenting the full triage, ask the user:

```
Phoenix — Quel tier attaquer ?
  [1] Tout corriger (🔴 → 🟡 → 🟢)
  [2] Corriger les 🔴 critiques (N items)
  [3] Corriger les 🟡 majeurs (N items)
  [4] Corriger les 🟢 mineurs (N items)
  [5] Pas maintenant
  [6] Convergence autonome (dry-run → plan unifié)
```

### Step 6b — Correction pipeline (audit → forge → commit → re-audit)

When the user chooses a tier (1-4) or convergence (6), corrections follow this pipeline:

```
For each triage item in the selected tier:
  1. FIX: edit code/config directly OR trigger the forge skill that can produce the fix
  2. GENERATE: if the item is about missing docs/diagrams/README → run the appropriate forge skill
  3. COMMIT: format the commit using cli-git-conventional (ghostwriter, zero AI markers)
  4. RE-AUDIT: re-run only the skills that sourced the fixed items
```

**Forge skills triggered automatically by audit findings:**

| Audit finding | Forge skill triggered | What it produces |
|--------------|----------------------|-----------------|
| Missing CONTRIBUTING.md, architecture docs | `cli-forge-doc` | CONTRIBUTING.md, docs/explanation/architecture.md |
| Missing or outdated README | `cli-forge-readme` | Updated README.md |
| Missing diagrams | `cli-forge-schema` | Mermaid diagrams in docs or README |
| CI/CD issues (missing stages, parallelism) | `cli-forge-pipeline` | Updated pipeline config |
| Code fixes, config fixes | Direct edit | Modified files |

**cli-git-conventional is ALWAYS used** — every correction commit goes through ghostwriter format. No exceptions, no AI trailers.

**Example correction sequence:**
```
Triage item #5: "Missing CONTRIBUTING.md" (source: cli-audit-doc)
  → cli-forge-doc generates CONTRIBUTING.md (absorbs CLAUDE.md content if present)
  → git add CONTRIBUTING.md
  → cli-git-conventional: "docs: add contributing guide with commit rules and git workflow"
  → cli-audit-doc re-runs → item resolved

Triage item #1: "Hardcoded password in Containerfile" (source: cli-audit-code)
  → Direct edit: remove hardcoded value, add env var
  → git add Containerfile entrypoint.sh
  → cli-git-conventional: "fix(security): externalize healthcheck password"
  → cli-audit-code re-runs → item resolved, may find cascade items
```

If the user chooses `[6]` Convergence autonome:
1. Read `references/convergence.md` for the full algorithm
2. Same pipeline but in an isolated worktree, fully autonomous
3. Track cascades (issues that only appear after earlier fixes)
4. Present a single unified plan with the complete diff
5. User approves (apply all), selects (cherry-pick), or rejects (discard)

### Step 7 — Phoenix Convergence (self-stopping)

The cycle stops automatically when **any** of these conditions is met:

| Condition | Meaning |
|-----------|---------|
| No 🔴 Tier 3 and no 🟡 Tier 2 items remain | Project is healthy — only cosmetic items left |
| Re-audit finds 0 new issues vs. previous pass | Fixes didn't introduce regressions — stable |
| User chooses `[5]` | Explicit stop |

When the cycle converges, output:

```
🔥 Phoenix — Cycle terminé

Passes effectuées : N
Issues résolues : X / Y
Score : avant → après

Le projet a convergé. Aucun défaut critique ou majeur restant.
```

### Step 8 — Output

- **Console** (always): full scorecard + complete 3-2-1 triage + phoenix choice
- **File** (if `docs/` exists): `docs/reviews/{YYYY-MM-DD}-audit.md` — includes full triage and convergence status

## Recurring Usage

```
/loop 7d /cli-cycle .
```

The skill automatically detects previous reports in `docs/reviews/` and shows progression.

## What this skill does NOT do

- **It does not generate from scratch.** It audits existing state
- **It does not run skills that don't apply.** No wasted tokens
- **It does not hide corrections.** Every finding is shown, classified, and actionable
