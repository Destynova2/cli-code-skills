---
name: cli-forge-doc
description: Generate and audit comprehensive project documentation from a Git repository. Produces standard documentation (CONTRIBUTING.md, architecture, troubleshooting) in Diataxis structure with zero AI markers by default. Use this skill whenever someone asks to document a project, generate docs, create a README, write API docs, or improve existing documentation. Also trigger when someone mentions "doc", "documentation", "readme", "explain this codebase", "onboard developers", or "make this project understandable".
argument-hint: "[git-repo-path]"
context: fork
agent: general-purpose
---

> **Optimization:** This skill uses on-demand loading. Heavy content lives in `references/` and is loaded only when needed.

> **Language rule:** Detect the project's primary language (from README, comments, docs, commit messages). Output your documentation in that language. If the project is bilingual, ask the user which language to use before proceeding.

# Doc Forge — Dual Documentation Generator & Auditor

Generate production-grade documentation for any Git project, optimized for both AI agents and human readers.

## Philosophy

Four principles stolen from the best, plus one operational rule:

1. **OpenBSD doctrine**: "Developers making a change to the system are expected to update the man pages along with their change." Docs ship with code. Man pages are authoritative. If it's not documented, it doesn't exist.
2. **Diátaxis framework**: Documentation has four modes — tutorials, how-to guides, reference, explanation. Never mix them.
3. **One doc, two readers**: Good documentation serves both humans and AI agents. Standard files (architecture.md, CONTRIBUTING.md, TROUBLESHOOTING.md) are naturally consumable by both — no special AI format needed.
4. **Single source of truth (the spliceosome principle)**: each fact has exactly one canonical home. Other documents that need to mention the fact must link to it, transclude from it, or be generated from it — never duplicate it by hand. A eukaryotic gene is transcribed once and the spliceosome cuts the relevant exons for each context; documentation should work the same way. One source, many splicings (README, CONTRIBUTING, --help, slide deck), zero hand-maintained copies. Duplicated Truth is the #1 cause of doc rot — every parallel copy drifts.
5. **Gotchas** — read `../../gotchas.md` before producing output to avoid known mistakes

## Input

`$ARGUMENTS` is the path to the Git repository (or current directory if empty).

If no argument: look for a Git repo in the current working directory.

## Step 0: Reconnaissance

Before writing a single line of doc, **read the project**. This is non-negotiable.

```
1. List the root directory
2. Read: README.md, CONTRIBUTING.md, Cargo.toml, package.json, pyproject.toml, go.mod, Makefile, Dockerfile, docker-compose.yml, .gitlab-ci.yml, .github/workflows/*, justfile
3. Identify: primary language(s), build system, CI/CD, test framework, deployment method
4. Sample 10-15 key source files (entry points, public API, core modules)
5. Read: docs/ directory if it exists
6. Check: git log --oneline -20 for recent activity focus
7. Check: git remote -v for repo URL
```

Build a mental model of:
- **What** the project does (purpose, problem it solves)
- **How** it's structured (architecture, key abstractions)
- **Who** uses it (library consumers, CLI users, operators, contributors)
- **Where** it runs (local, cloud, air-gapped, container)

## Step 1: Completeness Scan (The Checklist)

Before generating, score what exists. Use the **Documentation Completeness Index (DCI)**.

### DCI Formula

```
DCI = Σ(wᵢ × sᵢ) / Σ(wᵢ) × 10

Where:
  wᵢ = weight of item i (importance)
  sᵢ = score of item i (0.0 to 1.0)
```

### Checklist Items (with weights)

| # | Item | Weight | What to check |
|---|------|--------|---------------|
| 1 | Project overview (README or equivalent) | 5 | Exists, has purpose statement, not just boilerplate |
| 2 | Getting started / quickstart | 5 | Can a new dev go from clone to running in <10 min? |
| 3 | Architecture overview | 4 | High-level structure documented somewhere |
| 4 | API reference (public surface) | 5 | Doc comments on public items, generated docs |
| 5 | Configuration reference | 3 | All config options documented with defaults |
| 6 | Error handling guide | 3 | Error types, what they mean, how to fix |
| 7 | Deployment / operations guide | 3 | How to deploy, monitor, troubleshoot |
| 8 | Contributing guide | 2 | How to contribute, code style, PR process |
| 9 | Changelog / release notes | 2 | History of changes |
| 10 | License | 1 | Exists and is clear |
| 11 | CI/CD documentation | 2 | Pipeline stages, environments, secrets needed |
| 12 | Security documentation | 3 | Threat model, auth flows, secrets management |
| 13 | Troubleshooting / gotchas guide | 3 | Known issues, workarounds, common mistakes documented |
| 14 | Examples / tutorials | 4 | Working, tested, progressive complexity |
| 15 | Inline doc coverage (public API) | 4 | % of public items with doc comments |
| 16 | Cross-references & linking | 2 | Types linked, related items connected |

**Scoring guide**: 0.0 = absent, 0.25 = stub/placeholder, 0.5 = exists but incomplete, 0.75 = good but gaps, 1.0 = excellent.

Output the DCI score table and overall score before proceeding.

### Gap Analysis (ISO 13485 pattern)

Beyond raw scoring, run a **compliance gap analysis** based on project type:

| Project type | Required docs (gap analysis target) |
|--------------|-------------------------------------|
| Library | README, CHANGELOG, LICENSE, API_REFERENCE, CONTRIBUTING, SECURITY |
| CLI tool | README, --help, man page or equivalent, CHANGELOG, LICENSE, examples |
| Service | README, ARCHITECTURE, DEPLOYMENT, OPERATIONS, SECURITY, MONITORING, RUNBOOK |
| Infrastructure | README, ARCHITECTURE, NORMS_DEVIATIONS, OPERATIONS, TROUBLESHOOTING, CIS_AUDIT (if applicable) |

**Output:**
```
Gap Analysis — {project-type}
Compliance: 7/10 required docs present (70%)

Missing (prioritized by impact):
  1. SECURITY.md (high — required for production)
  2. RUNBOOK.md (medium — required for on-call)
  3. CHANGELOG.md (low — semantic-release may auto-generate)
```

## Step 2: Generate Project Documentation (Machine + Human Layer)

All project knowledge is distributed into **standard documentation files**. No AI-specific markers are generated by default.

**OPSEC rationale:** Files named `AGENTS.md`, `CLAUDE.md`, or `llms.txt` score DREAD 8.8/10 — they expose full architecture, credential patterns, and gotchas in files explicitly labeled for AI consumption. During a security audit (CIS, SOC 2, PCI-DSS, ISO 27001, defense contracts), these files signal AI involvement in development, raising accountability and supply chain concerns. Standard documentation files carry the same content with zero signal.

### Content redistribution map

All project context is placed in standard files. An AI agent reads these files naturally — no special format needed.

| Content | Standard destination |
|---------|---------------------|
| Stack, Architecture | `docs/explanation/architecture.md` |
| Domain Concepts | `docs/explanation/glossary.md` or `docs/reference/glossary.md` |
| Key Patterns, Conventions | `docs/explanation/architecture.md` (section "Conventions") |
| Commands | `README.md` (section "Usage" or "Commands") |
| Gotchas, known issues | `docs/TROUBLESHOOTING.md` |
| Commit rules, git workflow | `CONTRIBUTING.md` |
| CI/CD validation | `docs/reference/ci-cd.md` |

**Rules:**
- Describe capabilities, not file paths (paths change, capabilities don't)
- Domain concepts are stable — document them. File structure is volatile — don't
- Include package manager explicitly if non-default
- Never dump style guides — let the linter do the linter's job
- Use progressive disclosure: point to `docs/` for details, don't inline everything
- **Each fact has exactly one home (SSOT, the spliceosome rule):** if two generated files would describe the same configuration option, port number, version, install command, or architectural component, one must link to the other instead of restating it. When in doubt, the canonical home is the file most likely to be read first by the relevant audience (README for users, CONTRIBUTING for contributors, ARCHITECTURE for designers). Never generate Duplicated Truth — every parallel copy is a future drift.

### Cleanup of existing AI markers

If any AI-specific files exist in the project, absorb their content and delete them:

| File | Tool | Absorb into |
|------|------|-------------|
| `CLAUDE.md` | Claude Code | `CONTRIBUTING.md` |
| `GEMINI.md` | Gemini CLI | `CONTRIBUTING.md` |
| `AGENTS.md` | Codex/multi-tool | `docs/explanation/architecture.md` |
| `llms.txt`, `llms-full.txt` | Generic | `docs/index.md` |
| `.cursorrules`, `.cursor/rules/*.mdc` | Cursor | `CONTRIBUTING.md` (coding conventions) |
| `.windsurfrules`, `.windsurf/rules/*.md` | Windsurf | `CONTRIBUTING.md` |
| `.github/copilot-instructions.md` | Copilot | `CONTRIBUTING.md` |
| `.aider.conf.yml` | Aider | Discard (tool config, not project knowledge) |

After absorption:
1. **Delete** all AI marker files listed above
2. **Add** AI tool config dirs to `.gitignore` (`.cursor/`, `.windsurf/`, `.aider*`, `.claude/`, `.continue/`, `.junie/`)
3. **Warn** the user about AI trailers in git history — delegate to `cli-git-conventional --audit-markers` for cleanup options

### Opt-in: AI marker files (`--ai-markers` flag)

When explicitly requested with `--ai-markers`, additionally generate:

- `AGENTS.md` — flat LLM context file (< 300 lines) with Stack, Architecture, Domain Concepts, Key Patterns, Commands, Gotchas
- `llms.txt` — documentation index following llmstxt.org standard
- `llms-full.txt` — (optional, small projects) single concatenated file with all doc content inline

These files **duplicate** content already in the standard docs. They exist only for convenience when OPSEC is not a concern.

## Step 3: Generate Human Documentation (Diátaxis Layer)

This layer serves humans — from newcomers to experts. Follow the **Diátaxis framework** strictly: tutorials, how-to guides, reference, explanation. Never mix modes.

Read `references/diataxis.md` for the full directory structure, writing rules per quadrant, design doc template, and progressive complexity funnel.

## Step 4: Audit (Score the Result)

After generation, run the doc-audit framework on the result. See `reference.md` for the full audit methodology.

### Quick Audit Checklist

For each generated file, verify:

1. **Accuracy**: Does the doc match the actual code? (Read the implementation.)
2. **Completeness**: Are all public items covered?
3. **Freshness**: Does it reference current APIs, not stale ones?
4. **Examples**: Do examples compile/run? Do they use `?` not `unwrap()`?
5. **Cross-refs**: Are types and functions linked, not just named?
6. **Progressive**: Can a newcomer navigate from zero to competent?

### Documentation Debt Score

```
Doc Debt = (Public Items − Documented Items) / Public Items × 100

Thresholds:
  0-10%  → Green  (healthy)
  10-25% → Yellow (needs attention)
  25-50% → Orange (significant debt)
  50%+   → Red    (documentation emergency)
```

## Step 5: Output

### Files to Generate

At minimum, produce:
1. `CONTRIBUTING.md` — Commit rules, git workflow, PR process
2. `docs/explanation/architecture.md` — Full architecture, domain concepts, conventions
3. `docs/TROUBLESHOOTING.md` — Gotchas, known issues, workarounds
4. `docs/index.md` — Human landing page
5. `docs/tutorials/getting-started.md` — First-contact tutorial
6. `docs/reviews/DCI-REPORT.md` — Documentation completeness index with scores and recommendations

If the project warrants it (public API, library, complex system), also produce:
- Full Diátaxis docs/ tree
- Updated inline doc comments (as a diff or suggestions)
- `docs/design/000-template.md` — Design doc template for the team to reuse
- Design docs for major existing architectural decisions (if none exist yet)

### Presentation

Present the DCI score first, then the generated files. Summarize:
- Current state (DCI score)
- What was generated
- Top 3 highest-impact improvements still needed
- Recommended next steps (ordered by effort vs impact)

## Dynamic Handoffs

| Condition detected | Recommend | Why |
|-------------------|-----------|-----|
| Project has shell scripts undocumented | `/cli-audit-shell` | Assess script quality for doc accuracy |
| Architecture section reveals coupling concerns | `/cli-audit-tangle` | Topology audit |
| README needs rewrite | `/cli-forge-readme` | Focused README generation |

**Rule:** Recommend, don't auto-execute.

## Adaptation Rules

1. **Scale to project size**: A 200-line CLI script needs a good README, not a docs site. A platform with 50K LOC needs the full Diátaxis treatment.
2. **Match the language**: Follow the language's idiomatic doc conventions (see `reference.md`).
3. **Respect existing docs**: Don't overwrite good existing docs. Augment and cross-reference.
4. **Monorepo support**: If the project has multiple packages/services, generate per-package docs + a root-level overview.
5. **Air-gapped aware**: If the project targets air-gapped environments, emphasize offline-complete documentation. No external links that won't resolve.
6. **Confidence markers**: If you're unsure about something (inferred from code but not confirmed), mark it with `<!-- NEEDS-REVIEW: [reason] -->` rather than guessing.
