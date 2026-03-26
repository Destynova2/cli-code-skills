---
name: cli-audit-sync
description: "Verify documentation-code coherence: detect stale references, broken links, terminology drift, outdated diagrams, and non-working examples. Use when the user wants to check if docs match reality, detect doc drift, verify README accuracy, find stale documentation, or says 'is my doc up to date', 'check coherence', 'sync docs', 'doc drift', 'stale docs', 'verify documentation'. Also triggers on 'broken links', 'outdated references', 'docs match code'."
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

# Audit Sync — Documentation-Code Coherence Verifier

Detect every place where documentation says one thing and the code says another.

> "A stale diagram is worse than no diagram. A README with wrong commands is worse than no README."
> — Inspired by Swimm, DocPrism, and the Docs-as-Tests methodology

## Core Principle

**Documentation is a contract with the reader.** Every statement in docs is an implicit promise: "this is true right now." This skill finds broken promises.

**Read `../../gotchas.md`** before producing output to avoid known pitfalls.

---

## Three-Layer Verification

### Layer 1 — Structural (lychee-style)

Check that every reference in docs points to something real.

**What to check:**

| Check | How | Severity |
|-------|-----|----------|
| Internal file links (`[text](path/to/file)`) | Glob for the target file, verify it exists | Critical |
| Anchor links (`[text](#section)`) | Parse target file for matching heading | Warning |
| Image references (`![alt](path/to/image)`) | Verify image file exists | Critical |
| External URLs | Skip by default (network-dependent). Flag `[TODO]` or placeholder URLs | Info |
| Code block file references (`src/main.rs`, `config.yaml`) | Grep for mentioned filenames in the project | Warning |
| Import/use statements mentioned in docs | Verify the module/package/crate exists | Critical |

**How to run:**

1. Glob all documentation files: `**/*.md`, `docs/**/*`
2. Extract all links using regex: `\[.*?\]\((.*?)\)`, `!\[.*?\]\((.*?)\)`
3. For each link, verify the target exists relative to the doc file's location
4. Collect results as: `{file, line, link, status: found|broken|ambiguous}`

### Layer 2 — Semantic (DocPrism/Swimm-style)

Check that names, concepts, and claims in docs match the actual code.

**What to check:**

| Check | How | Severity |
|-------|-----|----------|
| Module/package names | Extract names from docs, grep in source tree | Critical |
| Function/method names | Extract `function_name()` patterns from docs, grep in code | Critical |
| Type/struct/class names | Extract PascalCase names from docs, grep in code | Critical |
| Endpoint paths (`/api/v1/users`) | Extract URL patterns from docs, grep in route definitions | Critical |
| CLI commands/flags | Extract backtick commands from docs, grep in CLI definition code | High |
| Environment variables | Extract `ENV_VAR` patterns from docs, grep in code | High |
| Config keys | Extract config references from docs, verify in config files | Warning |
| Dependency names | Extract package names from docs, verify in manifest (Cargo.toml, package.json...) | Warning |
| Version numbers | Extract versions from docs, compare with manifest/Cargo.toml | Warning |
| File counts / feature counts | Extract numeric claims ("5 endpoints", "3 modules"), count actual | Warning |

**Methodology (inspired by DocPrism's LCEF):**

1. **Local Categorization** — For each doc file, extract all code-referencing tokens:
   - Backtick-wrapped identifiers: `` `function_name` ``
   - PascalCase words in prose (likely type names)
   - UPPER_SNAKE_CASE words (likely constants/env vars)
   - Path-like strings (`src/`, `/api/`, `config/`)
   - Command-like strings starting with `$`, `./`, or common CLI tools

2. **External Filtering** — For each extracted token:
   - Search in source code (grep)
   - Search in manifest files
   - Search in config files
   - Classify: `found` | `not_found` | `renamed` (fuzzy match) | `generic` (skip common words)

3. **Drift Detection** — Check git blame/log for recently changed files:
   - If a source file was modified recently but its doc references weren't → flag as potential drift
   - If a doc file references a deleted file → flag as stale

**Terminology Consistency:**

Scan all docs for the same concept referred to differently:
- "user" vs "client" vs "account" vs "customer" for the same entity
- "service" vs "server" vs "backend" vs "API" for the same component
- Build a terminology map and flag inconsistencies

### Layer 3 — Executable (Docs-as-Tests-style)

Check that commands and examples in docs actually work.

**What to check:**

| Check | How | Severity |
|-------|-----|----------|
| Shell commands in README (`` ```bash ``) | Parse code blocks, classify as runnable vs illustrative | Critical |
| Installation commands | Verify the install path exists or the package is published | Critical |
| Build commands (`cargo build`, `npm install`) | Dry-run if safe (check syntax, not execute) | High |
| Example code snippets | For Rust: check if they'd compile (type/import references). For others: syntax check | Warning |
| Expected output shown in docs | If docs show `→ output`, verify format is plausible | Info |

**Safety rules:**
- **NEVER execute** commands that modify state (install, write, delete)
- **Dry-run only**: verify syntax, check that referenced files/tools exist
- **Classify commands** before checking:
  - `safe_to_verify`: `ls`, `cat`, `grep`, `--version`, `--help`
  - `check_syntax_only`: `cargo build`, `npm install`, `docker run`
  - `skip`: `rm`, `sudo`, `curl -X POST`, anything with side effects

**Mermaid diagram verification:**

For each Mermaid block in docs:
1. Parse node names from the diagram
2. Grep for those names in source code
3. Flag nodes that don't correspond to real files/modules/services
4. Flag edges (relationships) that don't match import/call patterns

---

## Workflow

### Mitosis — Scale verification to doc volume

| Signal | Tier |
|--------|------|
| 1-2 doc files (README + maybe CONTRIBUTING) | **S** |
| docs/ directory with 5-15 files | **M** |
| Full docs site (20+ files, API ref, tutorials) | **L** |

| Tier | Verification depth |
|------|-------------------|
| S | Layer 1 (structural links) only + quick Layer 2 scan |
| M | Layer 1 + Layer 2 (semantic) + sample Layer 3 (executable) |
| L | All 3 layers, full terminology map, diagram verification |

**Rule:** Don't run terminology mapping on a project with 1 README.

### Step 1 — Discover documentation

```
Glob: **/*.md, docs/**/*.*, README*, CHANGELOG*, CONTRIBUTING*
```

Also check for inline documentation:
- Doc comments in source code (`///`, `/** */`, `#`, `"""`)
- AGENTS.md, CLAUDE.md, llms.txt

### Step 2 — Discover code structure

Build a "source of truth" index:
- File tree (all source files)
- Exported symbols (public functions, types, modules)
- Route definitions (endpoints)
- Config keys (from config files)
- CLI commands/flags (from argument parser definitions)
- Dependencies (from manifest)
- Version (from manifest)

### Step 3 — Run all three layers

Launch layers in parallel via sub-agents:
- Agent 1: Layer 1 (structural)
- Agent 2: Layer 2 (semantic)
- Agent 3: Layer 3 (executable)

### Step 4 — Synthesize report

```markdown
# Doc-Code Sync Report — {project-name}

**Date:** {date}
**Files scanned:** {n} docs, {m} source files
**Coherence score:** {X}/10

## Summary

| Layer | Checks | Passed | Failed | Warnings |
|-------|--------|--------|--------|----------|
| Structural (links & refs) | 42 | 38 | 3 | 1 |
| Semantic (names & claims) | 67 | 60 | 5 | 2 |
| Executable (commands & examples) | 12 | 10 | 1 | 1 |
| **Total** | **121** | **108** | **9** | **4** |

## Critical Issues (fix now)

| # | File:Line | Issue | What docs say | What code says |
|---|-----------|-------|---------------|----------------|
| 1 | README.md:45 | Stale endpoint | `/api/v1/users` | Renamed to `/api/v2/accounts` |
| 2 | docs/arch.md:23 | Dead module ref | `auth_service` | Deleted in commit abc123 |
| 3 | README.md:12 | Wrong dep count | "5 dependencies" | Cargo.toml has 8 |

## Warnings (fix soon)

| # | File:Line | Issue | Details |
|---|-----------|-------|---------|
| 1 | docs/api.md:67 | Terminology inconsistency | "user" (3x) vs "account" (2x) for same entity |
| 2 | README.md:89 | Possibly stale | `config.yaml` mentioned but file is `config.toml` |

## What's in sync (good)

- README install commands reference correct paths
- All Mermaid diagram nodes match real modules
- Version in README matches Cargo.toml
- 38/42 internal links valid

## Terminology Map

| Concept | Terms used | Recommended | Where |
|---------|-----------|-------------|-------|
| The end user | user (12x), client (3x), account (1x) | `user` (most frequent) | README, docs/api.md |
```

### Step 5 — Output

- **Console**: summary table + critical issues
- **File** (if `docs/` exists): `docs/reviews/{YYYY-MM-DD}-sync.md`

---

## Scoring

| Score | Meaning |
|-------|---------|
| 10/10 | Every doc reference matches code. Zero drift |
| 8-9/10 | Minor warnings only (terminology, info-level) |
| 6-7/10 | Some stale references but no broken commands |
| 4-5/10 | Multiple broken references, outdated sections |
| 1-3/10 | Documentation is actively misleading |

---

## Integration with other cli-* skills

| Skill | How cli-audit-sync uses it |
|-------|--------------------------|
| `cli-audit-doc` | Scores doc **quality**. cli-audit-sync scores doc **accuracy** |
| `cli-forge-schema` | Generates diagrams. cli-audit-sync **verifies** existing diagrams match code |
| `cli-forge-readme` | Generates READMEs. cli-audit-sync **verifies** existing README matches reality |
| `cli-cycle` | Calls cli-audit-sync as part of the full project review |

---

## What this skill does NOT do

- **Does not fix docs** — it reports. Use `cli-forge-*` skills to fix
- **Does not execute unsafe commands** — dry-run verification only
- **Does not check external URLs** — no network calls (use `lychee` separately for that)
- **Does not replace linters** — Vale/markdownlint handle prose style, this handles truth
