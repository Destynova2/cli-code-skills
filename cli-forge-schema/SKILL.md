---
name: cli-forge-schema
description: "Use this skill to generate, convert, or refine diagrams and visual representations in GitHub-compatible Mermaid. Triggers include: 'diagram', 'schema', 'flowchart', 'sequence diagram', 'architecture diagram', 'ER diagram', 'state machine', 'gantt', 'mindmap', 'timeline', 'mermaid', 'visualize', 'draw', 'convert table to diagram', 'kanban', 'PERT', 'roadmap', 'changelog', 'sankey', 'quadrant', 'git graph', or any request to create, fix, or improve a visual representation of data, processes, or architecture. Also triggers when the user pastes broken Mermaid, asks to simplify a complex diagram, wants to convert markdown tables/bullet lists/SQL schemas/JSON structures/API lists/RACI matrices into visual formats, or says 'make this visual'. Do NOT use for actual image generation (PNG, SVG rendering) — output is always Mermaid markdown."
argument-hint: "[description-or-file-to-visualize]"
context: fork
agent: general-purpose
allowed-tools:
  - Read
  - Write
  - Grep
  - Glob
  - Bash
---

> **Language rule:** Detect the project's primary language (from README, comments, docs, commit messages). Output labels, titles, and annotations in that language. If the project is bilingual, ask the user which language to use before proceeding.

# Schema Forge — Clear Diagrams, Zero Syntax Errors

Generate professional, GitHub-compatible Mermaid diagrams that are **readable first, pretty second**.

## Core Philosophy

1. **Clarity over completeness** — a diagram that shows too much shows nothing. If it's complex, split it
2. **Right tool for the job** — pick the diagram type that makes the data obvious, not the one that looks fancy
3. **GitHub-first** — every diagram must render on GitHub without errors. No FontAwesome, no click events, no exotic features
4. **UX ergonomics** — treat diagrams like UI: if a user needs more than 5 seconds to understand the flow, redesign it
5. **Truth over aspiration** — a diagram must reflect what EXISTS in the code, not what someone hopes to build. Stale diagrams are worse than no diagrams

---

## Operating Modes

This skill operates in three modes, detected from the user's input:

### Mode A — Single Diagram

The user describes something specific to visualize. Generate one diagram.

**Trigger:** text description, broken Mermaid, table to convert, specific request.

### Mode B — Doc Scan & Convert

The user points at a directory or says "scan", "audit docs", "make visual". The skill crawls documentation files and converts everything that would be **genuinely more readable** as Mermaid.

**Trigger:** path to a directory, "scan docs", "convert everything", "make docs visual".

### Mode C — Project Schema

The user wants a diagram of the **current project** (architecture, module structure, data flow). The skill reads the actual codebase to generate accurate diagrams.

**Trigger:** "project schema", "architecture of this project", "visualize this codebase", "show me the structure".

---

## Mode B — Doc Scan & Convert

### Workflow

1. **Glob** for documentation files: `**/*.md`, `**/*.adoc`, `**/*.rst`, `docs/**/*`
2. **Read** each file and identify conversion candidates:

| Pattern found | Worth converting? | Convert to |
|--------------|------------------|-----------|
| Markdown table with relationships (FK, parent, depends) | Yes | `erDiagram` |
| Markdown table with steps/phases/status | Yes | `flowchart`, `stateDiagram-v2`, or `gantt` |
| Markdown table with proportions/percentages | Yes | `pie` |
| Numbered step-by-step instructions (> 4 steps) | Yes | `flowchart` |
| Nested bullet list (> 3 levels) | Yes | `mindmap` |
| Decision tree described in prose | Yes | `flowchart` with diamonds |
| State/lifecycle described in prose | Yes | `stateDiagram-v2` |
| Sequence of API calls described in prose | Yes | `sequenceDiagram` |
| Architecture described in prose | Yes | `flowchart` with subgraphs |
| Simple flat table (just key-value) | **No** — table is already clear | Skip |
| Short bullet list (< 4 items, 1 level) | **No** — already readable | Skip |
| Prose paragraph without structure | **No** — no visual equivalent | Skip |

3. **For each candidate**, generate the Mermaid block and propose it as a replacement
4. **Output a summary** listing:
   - File path
   - Line range of the original content
   - Proposed diagram type
   - The Mermaid block
   - Whether to replace inline or add below the original

**Key rule:** Only convert if the diagram is **genuinely more readable** than the original. A table with 3 rows and 2 columns is fine as a table. Don't convert for the sake of converting.

---

## Mode C — Project Schema

### Workflow

1. **Analyze the actual codebase** (not docs, not aspirations):
   - Read project manifest (`Cargo.toml`, `package.json`, `go.mod`, etc.)
   - Glob source files to understand module structure
   - Grep for key patterns: HTTP routes, DB models, message queues, external calls
   - Read entry points (`main.rs`, `main.go`, `index.ts`, `app.py`)

2. **Generate diagrams that reflect reality:**

| What to diagram | How to verify | Diagram type |
|----------------|--------------|-------------|
| Module/package structure | `ls` + manifest | `flowchart TB` with subgraphs |
| Dependencies between modules | `use`/`import` statements in code | `flowchart` with edges |
| API endpoints | Route definitions in code | Table + `sequenceDiagram` for key flows |
| Data models | Struct/class definitions, DB migrations | `erDiagram` or `classDiagram` |
| State machines | Enum variants + transition functions | `stateDiagram-v2` |
| Async/message flows | Channel/queue usage in code | `sequenceDiagram` |
| Deployment | Dockerfile, docker-compose, k8s manifests | `flowchart` with infra nodes |

3. **Tag each diagram with its source of truth:**

```markdown
<!-- Schema source: src/models/*.rs, migrations/*.sql -->
<!-- Generated by /cli-forge-schema — verify before committing -->
```

4. **Distinguish IMPLEMENTED vs PLANNED:**
   - If it exists in code → solid lines, normal style
   - If it exists only in docs/TODO → dashed lines + `classDef planned fill:#F8F9FA,stroke:#BDC3C7,stroke-dasharray: 5 5,color:#95A5A6`
   - If it exists in code but NOT in docs → flag as undocumented

5. **Accuracy checklist** (run before output):
   - [ ] Every node in the diagram corresponds to a real file, module, or endpoint
   - [ ] Every edge represents a real import, call, or data flow found in code
   - [ ] No aspirational components (not yet implemented) shown as solid
   - [ ] Planned/future items clearly marked with dashed style
   - [ ] Source files referenced in a comment above the diagram

---

## Mode A — Single Diagram

### Step 1 — Understand the intent

What does the user want to visualize?

| Input | Action |
|-------|--------|
| Text description ("show me how X works") | Design the right diagram type from scratch |
| Existing Mermaid block (broken or ugly) | Fix syntax errors, improve readability |
| Markdown table | Convert to the most appropriate visual format |
| Code/architecture | Extract structure and visualize it |
| Kanban / PERT / timeline data | Convert to proper Mermaid format |
| "Simplify this diagram" | Reduce complexity, split if needed |

### Step 2 — Choose the right diagram type

Think like a UX designer. Match the **data shape** to the **diagram type**:

| Data shape | Best diagram | When to switch |
|-----------|-------------|---------------|
| Process with steps | `flowchart` | > 15 nodes → split into sub-flows |
| Interactions between actors | `sequenceDiagram` | > 8 participants → group into sub-diagrams |
| Object/class structure | `classDiagram` | > 10 classes → split by domain/module |
| State transitions | `stateDiagram-v2` | > 12 states → use composite states or split |
| Data relationships | `erDiagram` | > 8 entities → split by bounded context |
| Project timeline | `gantt` | > 20 tasks → split by phase/milestone |
| Proportions/distribution | `pie` | > 7 slices → group small ones into "Other" |
| Hierarchy/brainstorm | `mindmap` | > 4 levels deep → flatten or split branches |
| Chronological events | `timeline` | > 15 events → split by era/phase |
| Flow volumes | `sankey` | Keep under 15 flows |
| Priority/comparison | `quadrantChart` | Always exactly 4 quadrants |
| Git workflow | `gitGraph` | Keep under 10 commits per branch shown |
| Requirements | `requirementDiagram` | > 10 requirements → split by category |

**Decision heuristic:** If you hesitate between two types, pick the one with fewer visual elements for the same information.

### Step 3 — Complexity check (the 5-second rule)

Before writing, estimate complexity:

```
nodes + edges = complexity score
```

| Score | Action |
|-------|--------|
| < 20 | Single diagram, go ahead |
| 20-40 | Consider grouping with subgraphs |
| 40-80 | Split into 2-3 diagrams with a navigation overview |
| > 80 | Mandatory split. Create an overview diagram + detail diagrams |

**Splitting strategy:**
1. Create a **Level 0 overview** (max 7 boxes, one per sub-diagram)
2. Each sub-diagram is self-contained with clear entry/exit points
3. Name sub-diagrams consistently: "Detail: [Topic]"
4. Reference between diagrams using consistent node names

### Step 4 — Write the diagram

Follow these rules strictly:

#### GitHub Compatibility (non-negotiable)

- Use ` ```mermaid ` fenced code blocks
- **NO** FontAwesome icons (`fa:fa-*`) — use emoji or text instead
- **NO** click events or callbacks
- **NO** tooltips or interactive features
- **NO** `%%{init: {"theme": "..."}}%%` with custom fonts (GitHub ignores them)
- Node IDs: start with a letter, use `camelCase` or `snake_case`
- Keep under **50 nodes** and **100 edges** per diagram (GitHub hard limits: 280 edges)
- Keep total Mermaid block under **2000 characters** for reliable rendering
- Test-worthy: the diagram must parse without errors

#### Readability Rules

1. **Direction matters:**
   - `TB` (top-bottom) for hierarchies and flows
   - `LR` (left-right) for timelines and sequences
   - `RL` for right-to-left language projects
2. **Labels are mandatory** — no unlabeled arrows. Every edge tells a story
3. **Node shapes convey meaning:**
   - `[text]` rectangle = process/action
   - `(text)` rounded = start/end
   - `{text}` diamond = decision
   - `[(text)]` cylinder = database/storage
   - `[[text]]` subroutine = external system
   - `>text]` flag = event/signal
4. **Subgraphs for grouping** — use them to reduce visual noise, not add it
5. **Max 3 levels of nesting** in subgraphs

#### Color & Styling (functional, not decorative)

Use `classDef` for semantic coloring. Never more than 5 classes per diagram.

```mermaid
classDef primary fill:#4A90D9,stroke:#2C6CB0,color:#fff
classDef success fill:#27AE60,stroke:#1E8449,color:#fff
classDef warning fill:#F39C12,stroke:#D68910,color:#fff
classDef danger fill:#E74C3C,stroke:#C0392B,color:#fff
classDef neutral fill:#ECF0F1,stroke:#BDC3C7,color:#2C3E50
```

**When to use color:**
- Highlight the happy path (green) vs error path (red)
- Distinguish system boundaries (colored subgraphs)
- Mark status: done/active/pending
- **Never** use color as the only differentiator — labels must stand alone

### Step 5 — Convert existing content

When converting markdown tables, kanban boards, PERT charts, or other formats:

#### Table → Diagram

| Table type | Convert to |
|-----------|-----------|
| Comparison table | `quadrantChart` or styled `flowchart` with columns |
| Status/phase table | `stateDiagram-v2` or `gantt` |
| Relationship table | `erDiagram` |
| Timeline/date table | `timeline` or `gantt` |
| Hierarchy table | `mindmap` or `flowchart TB` |
| Metrics/proportions | `pie` |

#### Kanban → Mermaid

Convert kanban columns to a flowchart with styled subgraphs:

```mermaid
flowchart LR
    subgraph todo["To Do"]
        t1[Task 1]
        t2[Task 2]
    end
    subgraph progress["In Progress"]
        p1[Task 3]
    end
    subgraph done["Done"]
        d1[Task 4]
    end
    todo --> progress --> done
    class t1,t2 neutral
    class p1 warning
    class d1 success
```

#### PERT → Mermaid

Convert PERT charts to `gantt` with dependencies or `flowchart` with durations on edges:

```mermaid
gantt
    title Project PERT
    dateFormat  YYYY-MM-DD
    section Phase 1
    Task A           :a, 2025-01-01, 5d
    Task B           :b, after a, 3d
    section Phase 2
    Task C           :c, after a, 7d
    Task D           :d, after b c, 4d
```

#### Bullet lists / nested hierarchies → Mindmap

```mermaid
mindmap
    root((Topic))
        Category A
            Item 1
            Item 2
        Category B
            Item 3
```

#### Numbered steps → Flowchart

Convert step-by-step instructions (1. do X, 2. do Y) into a linear flowchart with decision points where alternatives exist.

#### Changelog / release notes → Timeline

```mermaid
timeline
    title Releases
    2025-01 : v1.0 Initial release
    2025-03 : v1.1 Performance fixes
    2025-06 : v2.0 Major rewrite
```

#### Roadmap → Gantt

Convert roadmap items with dates/quarters into `gantt` sections. Use `crit` for critical milestones.

#### Pros/cons or comparison lists → Quadrant Chart

Map items on effort/impact, cost/value, or risk/reward axes using `quadrantChart`.

#### SQL CREATE TABLE → ER Diagram

Parse `CREATE TABLE` statements and convert columns, types, PKs, FKs into `erDiagram` entities with relationships inferred from foreign keys.

#### JSON / YAML structures → Class Diagram or Mindmap

- Flat config → `mindmap` (keys as branches, values as leaves)
- Nested objects with types → `classDiagram` (objects as classes, fields as attributes)

#### API endpoint lists → Sequence Diagram

Convert a list of endpoints (method, path, description) into `sequenceDiagram` showing the actor, API, and downstream services interactions.

#### Budget / resource allocation → Sankey

```mermaid
sankey-beta
    Revenue,Engineering,50
    Revenue,Marketing,25
    Revenue,Operations,15
    Revenue,R&D,10
```

#### Git branching strategy → Git Graph

Convert branch descriptions into `gitGraph` with commits, branches, and merges.

#### RACI / responsibility matrix → Flowchart with subgraphs

Group tasks by responsible party using subgraphs, color-code by RACI role (R=primary, A=warning, C=neutral, I=grey).

#### Cron / schedules → Gantt

Convert recurring schedules into `gantt` with repeating task blocks to visualize time allocation.

### Step 6 — Review checklist

Before delivering, verify:

- [ ] Renders on GitHub (no syntax errors)
- [ ] Passes the 5-second rule (intent is immediately clear)
- [ ] All edges are labeled (except trivial parent-child)
- [ ] Node count under 50 per diagram
- [ ] Colors are functional, not decorative (max 5 `classDef`)
- [ ] No GitHub-incompatible features (FA icons, click events, tooltips)
- [ ] Direction matches the data flow (TB for hierarchy, LR for timeline)
- [ ] Complex diagrams are split with an overview
- [ ] Labels match the project's language

---

## Diagram Type Quick Reference

Read `references/diagram-types.md` for detailed syntax and GitHub-tested examples of each diagram type.

## Anti-patterns

| Don't | Do instead |
|-------|-----------|
| 50+ nodes in one diagram | Split into overview + details |
| Rainbow colors | Max 5 semantic colors |
| Unlabeled edges | Every arrow tells what happens |
| `fa:fa-icon` in nodes | Use emoji or text labels |
| Deeply nested subgraphs (4+) | Flatten to 2-3 levels max |
| Diagram as decoration | Diagram as communication |
| Same diagram type for everything | Match type to data shape |
| Wall of text in nodes | Short labels, details in docs |
