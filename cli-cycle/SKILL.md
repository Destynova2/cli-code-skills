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

> **Language rule:** Detect the project's primary language (from README, comments, docs, commit messages). Output your report in that language. If the project is bilingual, ask the user which language to use before proceeding.

# Cycle — Continuous Improvement Orchestrator

Run every applicable `cli-*` skill on the current project, collect results, and deliver a single prioritized action plan.

## Core Principle

**You don't duplicate logic — you delegate.** Each sub-agent reads the real SKILL.md from `~/.claude/skills/cli-*/SKILL.md` and follows its instructions. You orchestrate, collect, and judge.

---

## Workflow

### Step 1 — Discover available skills

Read the skills directory to find all installed `cli-*` skills:

```
Glob: ~/.claude/skills/cli-*/SKILL.md
```

Parse each SKILL.md frontmatter to extract:
- `name` — skill identifier
- `description` — what it does (to decide if it's applicable)
- `argument-hint` — what arguments it expects

### Step 2 — Detect project context

Analyze the target project (default: current directory):

1. **Read manifest** (`Cargo.toml`, `package.json`, `go.mod`, `pyproject.toml`, etc.)
2. **Check for docs**: `README.md`, `docs/`, `*.md` files
3. **Check for schemas/diagrams**: existing Mermaid blocks in `.md` files
4. **Check for infra**: `Dockerfile`, `docker-compose.yml`, `*.tf`, `helmfile.yaml`
5. **Get project size**: file count, primary language(s)

### Step 3 — Decide which skills to run

Not all skills apply to every project. Match skills to what exists:

| Skill | Run if... |
|-------|----------|
| `cli-audit-code` | Source code files exist |
| `cli-audit-doc` | Documentation files exist (`.md`, doc comments) |
| `cli-audit-sync` | Both source code AND documentation exist (verifies coherence between them) |
| `cli-forge-readme` | `README.md` exists (audit mode: compare current vs ideal) |
| `cli-forge-tree` | Always (audit current structure) |
| `cli-forge-schema` | Existing Mermaid diagrams found OR architecture worth diagramming |
| `cli-forge-doc` | Skip (generation, not audit — unless no docs exist at all) |
| `cli-forge-arch` | Skip (generation, not audit — unless user explicitly asks) |
| `cli-forge-infra` | Infra files exist (`Dockerfile`, `*.tf`, `helmfile.yaml`, `k8s/`) |

### Step 4 — Launch sub-agents in parallel

For each applicable skill, launch a sub-agent with this prompt pattern:

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

Launch all applicable agents **in parallel** for speed.

### Step 5 — Collect and synthesize

Once all sub-agents return, produce a unified report:

#### 5a — Score card

```markdown
## Project Health — {project-name}

| Area | Score | Status | Skill |
|------|-------|--------|-------|
| Code Quality | 7/10 | Needs work | cli-audit-code |
| Doc-Code Sync | 6/10 | Drift detected | cli-audit-sync |
| Documentation | 8/10 | Good | cli-audit-doc |
| README | 6/10 | Outdated | cli-forge-readme |
| Project Structure | 9/10 | Excellent | cli-forge-tree |
| Diagrams | 4/10 | Missing | cli-forge-schema |
| Infrastructure | N/A | No infra files | cli-forge-infra |
| **Overall** | **6.8/10** | | |
```

#### 5b — Priority matrix

Cross-reference all findings and rank by **impact x effort**:

```markdown
## Top 5 Actions

| # | Action | Impact | Effort | Source |
|---|--------|--------|--------|--------|
| 1 | Add error handling in auth module | High | Low | cli-audit-code |
| 2 | Update README quickstart section | High | Low | cli-forge-readme |
| 3 | Add sequence diagram for payment flow | Medium | Low | cli-forge-schema |
| 4 | Split utils.rs into focused modules | Medium | Medium | cli-audit-code |
| 5 | Add doc comments to public API | Medium | Medium | cli-audit-doc |
```

**Ranking rules:**
- High impact + Low effort = **Do first** (quick wins)
- High impact + High effort = **Plan next**
- Low impact + Low effort = **Nice to have**
- Low impact + High effort = **Skip**

#### 5c — What's going well

Don't just report problems. Highlight what the project does right:

```markdown
## Strengths

- Clean module boundaries (cli-audit-code: 9/10 on Structure)
- Excellent naming conventions (cli-audit-code: 9/10 on Naming)
- README has working quickstart (cli-forge-readme: verified)
```

#### 5d — Skill suggestions

If during the audit you notice patterns that suggest a **new skill** would be valuable, mention it:

```markdown
## Skill Suggestions

- The project has 15+ API endpoints but no contract tests → consider a `cli-audit-api` skill
- Heavy use of state machines but no visualization → `cli-forge-schema` in Mode C would help
```

### Step 6 — Output

Write the report to:
- **Console** (always): summary scorecard + top 5 actions
- **File** (if project has a `docs/` directory): `docs/reviews/{YYYY-MM-DD}-audit.md`

If a previous audit exists in `docs/reviews/`, compare scores and show trends:

```markdown
## Trends

| Area | Previous | Current | Delta |
|------|----------|---------|-------|
| Code Quality | 6/10 | 7/10 | +1 |
| Documentation | 8/10 | 8/10 | = |
| README | 4/10 | 6/10 | +2 |
```

---

## Recurring Usage

To run weekly:

```
/loop 7d /cli-audit-all .
```

The skill automatically detects previous reports in `docs/reviews/` and shows progression.

---

## What this skill does NOT do

- **It does not fix anything.** It reports and prioritizes. The user decides what to act on.
- **It does not generate from scratch.** It audits existing state. Use individual `cli-forge-*` skills to generate.
- **It does not run skills that don't apply.** No infra files = no infra audit. No wasted tokens.
