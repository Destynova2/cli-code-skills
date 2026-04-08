---
name: cli-audit-tangle
description: >
  Detect spaghetti code and dependency cycles using graph theory, spectral analysis, and biomimetic patterns.
  Finds god functions, circular dependencies, dead code, suboptimal module boundaries,
  CI/CD pipeline deadlocks, and inefficient call patterns. Uses call graph topology (not just line-level metrics)
  to identify structural problems invisible to linters. Provides concrete fix patterns per domain.
  Use when the user says 'spaghetti', 'tangle', 'untangle', 'god function', 'god object',
  'circular dependency', 'cycle detection', 'call graph', 'module coupling', 'dead code',
  'code topology', 'complexity analysis', 'who calls who', 'démêler', 'couplage',
  'dépendances circulaires', 'fonction dieu', 'code mort', 'import cycle', 'ImportError',
  'pipeline deadlock', 'CI stuck', 'needs deadlock', 'everything depends on everything'.
  Also triggers on 'refactor structure', 'split module', 'too coupled', 'architecture debt',
  'dependency diagnostic', 'workflow cycle'.
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

## Biological & physical mechanisms

| Source | Mechanism | What it does in code analysis |
|--------|-----------|------------------------------|
| **Topoisomérase** (enzyme) | Cuts DNA strands at knot points, passes strands through, re-ligates | Finds the **optimal cut points** in tangled call graphs. Type I = minor refactor (extract function). Type II = major restructuring (split module) |
| **Fourmis de feu** (Solenopsis) | Form living rafts by linking bodies — each ant grips exactly 2 neighbors. If one ant grips 12, the raft collapses | Detects **god functions** — nodes that grip too many neighbors. The raft (codebase) is fragile at those points |
| **Impasse ferroviaire** | Two trains on a single track facing each other — neither can advance | Detects **CI/CD deadlocks** — job A needs B, job B needs A |
| **Polymère emmêlé** | Longer polymer chains = higher viscosity = slower system | Measures **codebase viscosity** — deep dependency chains = every change touches 10 files |
| **Méduse** | Tentacles don't tangle thanks to mucus (reduced friction) | **Interfaces** are the mucus — reduce coupling friction between modules |

> Read `references/analogies.md` for extended analogies and mental models.

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
- **CI/CD** (`.github/workflows/`, `.gitlab-ci.yml`): analyze pipeline dependency graph
- **Empty**: analyze the entire project (code + CI if present)

## Workflow

### Step 0 — Detect language and tools

| Language | Call graph tool | AST tool | Quick dependency graph command |
|----------|----------------|----------|-------------------------------|
| Rust | `cargo-call-stack` or LSP references | `ast-grep` | `cargo tree --duplicates` |
| Python | `pyan3`, `py-call-graph` | `ast-grep` | `pydeps src/ --max-bacon 3 --no-show --dot \| dot -Tsvg > deps.svg` |
| JS/TS | `madge` | `ast-grep` | `npx depcruise src --include-only "^src" --output-type dot \| dot -Tsvg > deps.svg` |
| Go | `go-callvis` | `ast-grep` | `go-callvis -group pkg ./...` |
| Other | LSP call hierarchy | `ast-grep` | `grep -r "^import\|^from\|^use\|^require" src/ \| grep -v test \| sort \| uniq` |

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

**Quick symptom hints:**
- `ImportError: cannot import name X from partially initialized module Y` → Python cycle (quasi-certain)
- CI job stuck "waiting" with free runners → `needs` deadlock
- Stack overflow at init → circular instantiation
- Build looping → hidden transitive dependency

**Qualify each cycle found:**

| Type | Definition | Urgency |
|------|-----------|---------|
| **Hard cycle** | A imports B which imports A at module level | Mandatory fix — runtime breaks |
| **Strong coupling** | A and B know each other mutually but not at import level | Recommended — works but fragile |
| **God module** | One module imported by 50+ others, single point of failure | Strategic — split progressively |

**3c — Module boundary analysis (Fiedler vector)**

If the project has module structure:
1. Build inter-module call graph (edges between modules)
2. Compute coupling (calls between modules) vs cohesion (calls within modules)
3. Flag modules where coupling > cohesion (they shouldn't be separate)
4. Flag modules where internal clusters suggest a split (Fiedler cut)

**3d — Dead node detection (with competing hypotheses)**

Functions with in-degree = 0 AND not an entry point (main, test, public API) = potentially dead code.

**Mandatory: enumerate ≥3 competing hypotheses before flagging** (inspired by scientific hypothesis-generation):

| Hypothesis | Disproof condition |
|-----------|-------------------|
| H1: truly dead code | None of H2-H5 hold |
| H2: called via reflection (`getattr`, `__import__`, dynamic dispatch) | Grep for the function name as a string literal in the codebase |
| H3: public API for external consumers | Function is `pub` in `lib.rs` root, or in `__all__`, or in `mod.exports` |
| H4: called from tests, examples, benches (excluded from main scan) | Re-scan with test/example/bench dirs included |
| H5: called via macro expansion or trait dispatch | Grep for function name in macro definitions; check trait impls |

**Only flag as dead if ALL 5 hypotheses are refuted.** A finding without disproof is downgraded to "POTENTIALLY DEAD — confirm manually."

Read `references/analysis-methods.md` for detailed algorithms.

### Step 3e — Detect anti-patterns

Read `references/anti-patterns.md` for the 8 named anti-patterns with detection heuristics. Flag each with severity and evidence.

### Step 3f — CI/CD pipeline analysis (if applicable)

If `.github/workflows/` or `.gitlab-ci.yml` exists:

1. Extract the `needs`/`dependencies` graph from all workflow files
2. Build adjacency list: `job_A → [job_B, job_C]`
3. Run cycle detection (Tarjan) on the job graph
4. Detect:
   - **Deadlocks**: circular `needs` chains
   - **Bottleneck jobs**: single job that blocks > 50% of downstream jobs
   - **Redundant dependencies**: `needs` that are already transitively satisfied
   - **Workflow trigger cycles**: `workflow_run` chains that loop

**GitHub Actions detection:**
```bash
grep -r "needs:" .github/workflows/ | grep -v "^#"
grep -r "workflow_run\|workflow_call" .github/workflows/
```

**GitLab detection:**
```bash
grep -A5 "needs:\|dependencies:" .gitlab-ci.yml | grep -v "^--$"
```

### Step 4 — Classify refactoring recommendations

| Type | Biology | Trigger | Recommendation |
|------|---------|---------|---------------|
| **Type I** (minor) | Topoisomérase I — cut one strand | God function, high complexity | Extract sub-functions, reduce responsibility |
| **Type II** (major) | Topoisomérase II — cut both strands | Module coupling > cohesion, cycles between modules | Split module, reorganize architecture |
| **Cleanup** | Apoptose — programmed cell death | Dead functions (in-degree = 0) | Remove after confirmation |
| **Decouple** | Break ant raft overload | Circular dependencies | Introduce trait/interface, dependency inversion |
| **Unblock** | Railway switch | CI/CD deadlock, bottleneck job | Extract common upstream job, flatten `needs` graph |

Read `references/fix-patterns.md` for concrete remediation patterns by language and domain.

### Diagnostic checklist (quick self-check before reporting)

```
[ ] Graph built? (no eyeball diagnosis)
[ ] Cycles identified? (SCC located, not just "seems coupled")
[ ] Each cycle qualified? (hard / strong / god module)
[ ] Cut point identified? (interface / event / extraction)
[ ] Fix verifiable? (test, build, pipeline green)
```

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

## CI/CD Pipeline Topology (if applicable)

| Metric | Value | Status |
|--------|-------|--------|
| Workflow files | N | — |
| Total jobs | N | — |
| `needs` edges | N | — |
| Deadlocks (cycles) | N | {OK / Critical} |
| Bottleneck jobs (> 50% downstream) | N | {OK / Warning} |
| Max pipeline depth | N | {OK / Warning} |

| Issue | Jobs | Type | Recommendation |
|-------|------|------|---------------|
| Deadlock | `build-a ↔ test-b` | Cycle | Extract common `prepare` job |
| Bottleneck | `build` blocks 8/10 jobs | Hub | Parallelize with matrix strategy |

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
  - (ci_deadlock_count × 15)
  - (ci_bottleneck_count × 5)
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
| `cli-forge-pipeline` | Generates CI/CD pipelines. tangle audits their **dependency topology** |
| `cli-cycle` | Should call cli-audit-tangle as part of the full project review |

## Dynamic Handoffs

After your analysis, recommend these skills if conditions are met:

| Condition detected | Recommend | Why |
|-------------------|-----------|-----|
| CI/CD deadlocks or bottleneck jobs | `/cli-forge-pipeline` | Pipeline optimization with parallelism patterns |
| Call graph worth visualizing (> 20 nodes) | `/cli-forge-schema` | Generate Mermaid diagram of the topology |
| God functions with low code quality (LOC > 100, deep nesting) | `/cli-audit-code` on those files | Deep quality check on the most critical code |
| Module boundaries suggest architectural redesign | `/cli-forge-hld` | High-level design for the new structure |
| Shell scripts tangled (sourcing chains, circular includes) | `/cli-audit-shell` | Shell-specific structural analysis |

**Rule:** Recommend, don't auto-execute. Phrase as: "Consider running `/cli-forge-pipeline` — 2 CI deadlocks detected in the `needs` graph."

## What this skill does NOT do

- **Does not run code** — static analysis only
- **Does not fix code** — it reports topology and recommends cuts
- **Does not replace profilers** — flamegraph finds hot paths at runtime, tangle finds structural knots at rest
- **Does not check code quality** — naming, DRY, style are cli-audit-code's domain
