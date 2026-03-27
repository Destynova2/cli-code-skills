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

# Cycle — Continuous Improvement Orchestrator

Run every applicable `cli-*` skill on the current project, collect results, and deliver a single prioritized action plan.

## Core Principle

**You don't duplicate logic — you delegate.** Each sub-agent reads the real SKILL.md and follows its instructions. You orchestrate, collect, and judge.

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

**Wave 1 (foundation, parallel):** code, doc, test, tree, readme — independent scans
**Wave 2 (cross-cutting, parallel):** sync, drift, pipeline, schema, infra — use Wave 1 context

Wave 2 skills get Wave 1 summaries injected into their prompts for richer analysis.

### Step 5 — Collect and synthesize

Produce a unified report with:

1. **Score card** — all skill scores in one table
2. **Top 5 actions** — ranked by impact × effort (high impact + low effort = do first)
3. **Strengths** — what the project does right
4. **Skill suggestions** — patterns that suggest a new skill would help
5. **Trends** — if previous audit exists in `docs/reviews/`, compare scores

Read `references/orchestration.md` for report templates and ranking rules.

### Step 6 — Output

- **Console** (always): summary scorecard + top 5 actions
- **File** (if `docs/` exists): `docs/reviews/{YYYY-MM-DD}-audit.md`

## Recurring Usage

```
/loop 7d /cli-cycle .
```

The skill automatically detects previous reports in `docs/reviews/` and shows progression.

## What this skill does NOT do

- **It does not fix anything.** It reports and prioritizes
- **It does not generate from scratch.** It audits existing state
- **It does not run skills that don't apply.** No wasted tokens
