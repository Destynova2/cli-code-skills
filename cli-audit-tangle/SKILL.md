---
name: cli-audit-tangle
description: >
  Detect spaghetti code using graph theory, spectral analysis, and biomimetic patterns.
  Finds god functions, circular dependencies, dead code, suboptimal module boundaries,
  and inefficient call patterns. Uses call graph topology (not just line-level metrics)
  to identify structural problems invisible to linters.
  Use when the user says 'spaghetti', 'tangle', 'untangle', 'god function', 'god object',
  'circular dependency', 'cycle detection', 'call graph', 'module coupling', 'dead code',
  'code topology', 'complexity analysis', 'who calls who', 'démêler', 'couplage',
  'dépendances circulaires', 'fonction dieu', 'code mort'.
  Also triggers on 'refactor structure', 'split module', 'too coupled', 'architecture debt'.
argument-hint: "[file-or-directory-or-module]"
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

# Audit Tangle — Code Topology Untangler

Detect structural spaghetti using call graph analysis, spectral graph theory, and network physics. Goes beyond line-level linters to find **topological** problems: god functions, circular dependencies, dead nodes, and suboptimal module cuts.

> "La Topoisomérase ne voit pas le contenu de l'ADN — elle voit sa **forme**.
> Un nœud est un nœud, quel que soit le gène qu'il porte."

## Two biological mechanisms

| Organism | Mechanism | What it does in code analysis |
|----------|-----------|------------------------------|
| **Topoisomérase** (enzyme) | Cuts DNA strands at knot points, passes strands through, re-ligates | Finds the **optimal cut points** in tangled call graphs. Type I = minor refactor (extract function). Type II = major restructuring (split module) |
| **Fourmis de feu** (Solenopsis) | Form living rafts by linking bodies — each ant grips exactly 2 neighbors. If one ant grips 12, the raft collapses | Detects **god functions** — nodes that grip too many neighbors. The raft (codebase) is fragile at those points |

## Three mathematical lenses

| Lens | Source | What it reveals |
|------|--------|----------------|
| **Fiedler vector** | Spectral graph theory | The optimal 2-way partition of the call graph — where to cut to minimally disrupt |
| **Network resistance** | Electrical circuit theory | God functions = low effective resistance to all other nodes = current flows through them for everything |
| **Cycle detection** | Graph theory (Tarjan/Johnson) | Circular call chains — A→B→C→A — that create hidden coupling |

Read `references/analysis-methods.md` for detailed algorithms and interpretation.

## Mitosis — Scale analysis to project size

| Signal | Tier | Analysis depth |
|--------|------|---------------|
| < 50 functions | **S** | Full graph + metrics + cycles. Skip Fiedler (too small) |
| 50-200 functions | **M** | Full graph + metrics + cycles + Fiedler per module |
| 200-1000 functions | **L** | Sample hot modules (highest coupling), full cycle detection, Fiedler on top-level |
| > 1000 functions | **XL** | Module-level graph only (not function-level), sample god functions in top 5 modules |

Read `references/analysis-methods.md` for threshold calibration per tier.

## Input

`$ARGUMENTS` is the target to analyze:

- **File** (`src/core/mod.rs`): analyze that file's functions and their call relationships
- **Directory** (`src/`): analyze all functions in scope
- **Module name** (`auth`, `parser`): focus on that module's internal + external calls
- **Empty**: analyze the entire project

## Workflow

### Step 0 — Detect language and tools

| Language | Call graph tool | AST tool | Mutation tool |
|----------|----------------|----------|---------------|
| Rust | `cargo-call-stack` or LSP references | `ast-grep` | `cargo-mutants` |
| Python | `pyan3`, `py-call-graph` | `ast-grep` | `mutmut` |
| JS/TS | `madge` | `ast-grep` | `Stryker` |
| Go | `go-callvis` | `ast-grep` | `go-mutesting` |
| Other | LSP call hierarchy | `ast-grep` | — |

Check if tools are installed. If not, fall back to **LSP-based analysis** (grep for function definitions + call sites).

### Step 1 — Build call graph

1. Extract all function/method definitions in scope
2. For each function, find all call sites (what it calls, what calls it)
3. Build adjacency matrix: `M[i][j] = 1` if function `i` calls function `j`
4. Count: nodes (functions), edges (calls), density

**Fallback if no call graph tool:** Use grep/LSP to build the graph manually:
- Glob for function definitions
- For each function, grep for its name in all other files
- Build the adjacency list

### Step 2 — Compute metrics per function

For each function node, compute:

| Metric | What it measures | How |
|--------|-----------------|-----|
| **In-degree** | How many functions call this one | Count incoming edges |
| **Out-degree** | How many functions this one calls | Count outgoing edges |
| **Betweenness centrality** | How often this node is on shortest paths between others | Standard algorithm |
| **Cyclomatic complexity** | Internal branching complexity | Count branches (if/match/loop) |
| **Lines of code** | Size | Count |

Read `references/analysis-methods.md` for threshold calibration.

### Step 3 — Topological analysis

**3a — God function detection (Fourmis de feu)**

A god function = high in-degree + high out-degree + high betweenness centrality.

```
god_score = normalize(in_degree) × 0.3
          + normalize(out_degree) × 0.3
          + normalize(betweenness) × 0.4
```

Flag functions with `god_score > 0.7`.

**3b — Cycle detection (Topoisomérase — cut sites)**

Find all strongly connected components (Tarjan's algorithm).
Each SCC with > 1 node = a circular dependency.

**3c — Module boundary analysis (Fiedler vector)**

If the project has module structure:
1. Build inter-module call graph (edges between modules)
2. Compute coupling (calls between modules) vs cohesion (calls within modules)
3. Flag modules where coupling > cohesion (they shouldn't be separate)
4. Flag modules where internal clusters suggest a split (Fiedler cut)

**3d — Dead node detection**

Functions with in-degree = 0 AND not an entry point (main, test, public API) = potentially dead code.

Read `references/analysis-methods.md` for detailed algorithms.

### Step 3e — Detect anti-patterns

Read `references/anti-patterns.md` for the 8 named anti-patterns with detection heuristics. Flag each with severity and evidence.

### Step 4 — Classify refactoring recommendations

| Type | Biology | Trigger | Recommendation |
|------|---------|---------|---------------|
| **Type I** (minor) | Topoisomérase I — cut one strand | God function, high complexity | Extract sub-functions, reduce responsibility |
| **Type II** (major) | Topoisomérase II — cut both strands | Module coupling > cohesion, cycles between modules | Split module, reorganize architecture |
| **Cleanup** | Apoptose — programmed cell death | Dead functions (in-degree = 0) | Remove after confirmation |
| **Decouple** | Break ant raft overload | Circular dependencies | Introduce trait/interface, dependency inversion |

### Step 5 — Score and report

## Output Format

```markdown
# Tangle Audit — {project-name}

**Date:** {date}
**Scope:** {what was analyzed}
**Functions analyzed:** {N}
**Call edges:** {M}
**Graph density:** {density}%
**Tangle Score:** {X}/100 (lower = more tangled)

## Topology Summary

| Metric | Value | Status |
|--------|-------|--------|
| God functions (score > 0.7) | N | {OK / Warning / Critical} |
| Circular dependencies (SCCs) | N cycles | {OK / Warning / Critical} |
| Dead functions (in-degree = 0) | N | {Info} |
| Module coupling ratio | X:Y (coupling:cohesion) | {OK / Warning / Critical} |
| Max call chain depth | N | {OK / Warning / Critical} |

## God Functions (Fourmis de feu)

| Function | In° | Out° | Betweenness | LOC | God Score | Action |
|----------|-----|------|-------------|-----|-----------|--------|
| `process_request()` | 23 | 15 | 0.82 | 340 | 0.91 | Type II: split |
| `validate()` | 12 | 8 | 0.71 | 180 | 0.74 | Type I: extract |

## Circular Dependencies (Topoisomérase cut sites)

| Cycle | Functions involved | Cut recommendation |
|-------|-------------------|-------------------|
| SCC-1 | `a() → b() → c() → a()` | Break at `c→a`: introduce trait |
| SCC-2 | `parse() ↔ validate()` | Merge or extract shared logic |

## Module Boundaries (Fiedler analysis)

| Module | Cohesion | Coupling | Ratio | Recommendation |
|--------|----------|----------|-------|---------------|
| `auth` | 45 calls | 12 calls | 3.8:1 | OK |
| `core` | 20 calls | 35 calls | 0.6:1 | Type II: split |

## Dead Functions (Apoptose candidates)

| Function | File | Last modified | Confidence |
|----------|------|---------------|-----------|
| `old_handler()` | `src/api.rs:142` | 6 months ago | High |

## Anti-Patterns Detected

| # | Pattern | Severity | Evidence | Recommendation |
|---|---------|----------|----------|---------------|
| 1 | God Function | Critical | `process_request()`: god_score 0.91 | Type I: extract 3 sub-functions |
| 2 | Circular Dep | Critical | `parse ↔ validate` across modules | Introduce `Validator` trait |

## Refactoring Roadmap

| # | Action | Type | Impact | Effort | Target |
|---|--------|------|--------|--------|--------|
| 1 | Split `process_request()` | Type II | High | Medium | `src/core.rs` |
| 2 | Break `parse↔validate` cycle | Decouple | High | Low | `src/parser.rs` |
| 3 | Remove `old_handler()` | Cleanup | Low | Low | `src/api.rs` |
```

## Scoring

```
tangle_score = 100
  - (god_function_count × 10)
  - (cycle_count × 15)
  - (modules_with_coupling_gt_cohesion × 10)
  - (max_chain_depth > 10 ? 10 : 0)
  - (dead_function_ratio > 0.1 ? 5 : 0)
  clamped to [0, 100]
```

| Score | Verdict |
|-------|---------|
| 80-100 | Clean — well-structured, minimal tangling |
| 60-79 | Acceptable — some god functions, minor coupling |
| 40-59 | Tangled — structural issues need attention |
| 20-39 | Spaghetti — major restructuring needed |
| 0-19 | Critical — untestable, unmaintainable |

## Rules for the Auditor

1. **Topology over content.** This skill sees the shape of the call graph, not the quality of the code inside functions. That's cli-audit-code's job.
2. **Graph density matters.** A 500-function project with 200 edges is sparse (good). With 5000 edges is dense (bad). Calibrate thresholds to project size.
3. **Entry points are not god functions.** `main()`, test functions, and public API endpoints naturally have high degree — exclude them from god function analysis.
4. **Cycles between modules ≠ cycles within modules.** Inter-module cycles are Type II (restructure). Intra-module cycles are often fine (mutual recursion).
5. **Dead code needs confirmation.** In-degree = 0 might mean: called via macro, reflection, FFI, or dynamic dispatch. Flag but don't assert.
6. **Gotchas** — read `../../gotchas.md` before producing output to avoid known mistakes.

## Integration with other cli-* skills

| Skill | Relationship |
|-------|-------------|
| `cli-audit-code` | Scores code **quality** (naming, DRY). tangle scores code **topology** |
| `cli-audit-drift` | Checks behavioral conformity. tangle checks **structural** conformity |
| `cli-forge-schema` | Can visualize the call graph as a Mermaid diagram |
| `cargo-machete` | Finds unused **dependencies**. tangle finds unused **functions** |
| `runtime-perf-audit` | Finds **runtime** inefficiency. tangle finds **structural** inefficiency |
| `cli-cycle` | Should call cli-audit-tangle as part of the full project review |

## What this skill does NOT do

- **Does not run code** — static analysis only
- **Does not fix code** — it reports topology and recommends cuts
- **Does not replace profilers** — flamegraph finds hot paths at runtime, tangle finds structural knots at rest
- **Does not check code quality** — naming, DRY, style are cli-audit-code's domain
