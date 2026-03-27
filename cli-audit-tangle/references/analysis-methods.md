# Analysis Methods — Algorithms, Thresholds & Calibration

> **When to read:** During Steps 2-4 when computing metrics, running topological analysis, and classifying recommendations.

---

## God Function Detection — Fourmis de feu

### Algorithm

1. For each function, compute:
   - `in_degree` = number of functions that call it
   - `out_degree` = number of functions it calls
   - `betweenness` = fraction of shortest paths that pass through it

2. Normalize each metric to [0, 1] across all functions in scope.

3. Compute god score:
   ```
   god_score = normalize(in_degree) × 0.3
             + normalize(out_degree) × 0.3
             + normalize(betweenness) × 0.4
   ```

4. Betweenness is weighted higher because it captures **structural centrality**, not just popularity.

### Exclusions

Do NOT flag as god functions:
- `main()`, `run()`, `start()` — entry points
- Test functions (`#[test]`, `test_*`)
- Public API surface (trait implementations, handler functions)
- Dispatcher/router functions (their job IS to call many things)

### Threshold calibration by project size

| Project size | Functions | God score threshold | Max acceptable gods |
|-------------|-----------|-------------------|-------------------|
| **Small** | < 50 | 0.80 | 1-2 |
| **Medium** | 50-200 | 0.70 | 3-5 |
| **Large** | 200-1000 | 0.65 | 5-8 |
| **Very large** | > 1000 | 0.60 | 8-12 |

---

## Cycle Detection — Topoisomérase cut sites

### Algorithm

Use Tarjan's algorithm to find Strongly Connected Components (SCCs):
1. DFS traversal with lowlink tracking
2. Each SCC with > 1 node = a cycle
3. Rank cycles by size (more nodes = harder to break)

### Classification

| Cycle type | Description | Severity | Action |
|-----------|-------------|----------|--------|
| **Mutual recursion** | 2 functions call each other | Low (often intentional) | Verify intentional |
| **Intra-module cycle** | 3+ functions in same module | Medium | Consider extracting shared logic |
| **Inter-module cycle** | Functions in DIFFERENT modules call each other cyclically | Critical | Type II refactor: dependency inversion |
| **Transitive cycle** | A→B→C→...→A across > 5 nodes | Critical | Find the weakest link and cut |

### Cut point selection (Topoisomérase)

For each cycle, recommend WHERE to cut:
1. Find the edge with the **lowest call frequency** (fewest actual calls) — cutting there disrupts the least
2. If all edges have similar frequency, cut the edge between **different modules** (prefer inter-module cuts)
3. The cut point should introduce a **trait/interface** (dependency inversion), not just remove the call

---

## Module Boundary Analysis — Fiedler vector

### Algorithm

1. Build inter-module adjacency matrix:
   - `M[i][j]` = number of calls from module `i` to module `j`

2. Compute the Laplacian matrix:
   - `L = D - M` where `D` is the degree matrix

3. Find the Fiedler vector (eigenvector of the 2nd smallest eigenvalue of L):
   - Sign changes in the Fiedler vector = optimal partition points
   - Functions with values near 0 are "boundary" functions — they belong to both sides

4. Compare the Fiedler partition against actual module boundaries.

### Interpretation

| Finding | Meaning | Action |
|---------|---------|--------|
| Fiedler cut matches module boundary | Modules are well-defined | None |
| Fiedler cut is INSIDE a module | Module should be split | Type II: split at Fiedler cut |
| Fiedler cut crosses multiple modules | Modules should be merged or reorganized | Type II: restructure |
| Many functions near Fiedler value 0 | Ambiguous boundary — high coupling | Investigate coupling source |

### Coupling vs Cohesion ratio

For each module:
```
cohesion = calls within the module (internal edges)
coupling = calls to/from other modules (external edges)
ratio = cohesion / coupling
```

| Ratio | Verdict |
|-------|---------|
| > 3.0 | Excellent — highly cohesive, loosely coupled |
| 2.0-3.0 | Good |
| 1.0-2.0 | Acceptable — some coupling |
| 0.5-1.0 | Warning — more coupling than cohesion |
| < 0.5 | Critical — module boundaries are wrong |

---

## Network Resistance — God function confirmation

### Algorithm

Model the call graph as an electrical circuit:
- Each edge = a resistor (resistance = 1 / call_frequency, or 1 if frequency unknown)
- Compute **effective resistance** between each pair of nodes

A god function has **low effective resistance to many other nodes** = current flows through it for everything.

### Simplified approach (without full matrix inversion)

Use **closeness centrality** as a proxy for low resistance:
```
closeness(v) = (N-1) / Σ d(v, u)  for all u ≠ v
```
where `d(v, u)` is the shortest path length.

Functions with closeness > 0.7 AND god_score > threshold = confirmed god functions.

---

## Dead Code Detection

### Algorithm

1. Find all functions with `in_degree = 0` (nothing calls them)
2. Filter out entry points:
   - `main()`, `#[test]` functions, `#[tokio::main]`
   - Public API functions (pub fn in lib.rs root)
   - Trait implementations (may be called via dynamic dispatch)
   - Functions with `#[no_mangle]`, `extern "C"` (FFI)
   - Functions referenced in macros (grep for function name in macro invocations)

3. For remaining candidates, check git history:
   - Last modified > 6 months ago + in_degree = 0 → **High confidence dead**
   - Last modified < 1 month ago → **Low confidence** (maybe WIP)

### Confirmation via mutation testing

If `cargo-mutants` is available:
1. Delete the candidate function body (replace with `todo!()`)
2. If all tests pass → confirmed dead code
3. If tests fail → not dead, just not called in the static graph (dynamic dispatch, macros, etc.)

---

## Comparative Benchmarks

### Healthy codebase indicators

| Metric | Healthy | Warning | Critical |
|--------|---------|---------|----------|
| God functions (% of total) | < 2% | 2-5% | > 5% |
| Circular dependencies | 0 inter-module | 1-2 inter-module | > 2 inter-module |
| Dead functions (% of total) | < 5% | 5-15% | > 15% |
| Module coupling ratio (avg) | > 2.0 | 1.0-2.0 | < 1.0 |
| Max call chain depth | < 8 | 8-15 | > 15 |
| Graph density | < 5% | 5-15% | > 15% |

### Benchmarks by project type

| Project type | Typical tangle score | Target |
|-------------|---------------------|--------|
| CLI tool (< 5K LOC) | 75-90 | 80+ |
| Library (< 20K LOC) | 70-85 | 75+ |
| Web service (20-100K LOC) | 55-75 | 65+ |
| Enterprise monolith (> 100K LOC) | 35-60 | 50+ |
| Legacy codebase | 20-45 | 40+ |
