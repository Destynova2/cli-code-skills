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

Launch all applicable agents **in parallel** for speed.

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
| Semantic Drift | N/A | No CONTRACTS.md | cli-audit-drift |
| README | 6/10 | Outdated | cli-forge-readme |
| Project Structure | 9/10 | Excellent | cli-forge-tree |
| Diagrams | 4/10 | Missing | cli-forge-schema |
| Pipeline | 7/10 | Good | cli-forge-pipeline |
| Infrastructure | N/A | No infra files | cli-forge-infra |
| **Overall** | **6.8/10** | | |
```

### Priority matrix example

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

### Ranking rules

- High impact + Low effort = **Do first** (quick wins)
- High impact + High effort = **Plan next**
- Low impact + Low effort = **Nice to have**
- Low impact + High effort = **Skip**

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
