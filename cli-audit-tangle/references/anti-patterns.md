# Tangle Anti-Patterns — Named Patterns & Detection

> **When to read:** After Step 3 analysis, to match findings against named anti-patterns.

---

## 8 Structural Anti-Patterns

| # | Anti-Pattern | Biology/Physics | Detection | Severity |
|---|-------------|----------------|-----------|----------|
| 1 | **God Function** | Fourmi de feu agrippant 12 voisines | god_score > threshold, in° + out° + betweenness all high | Critical |
| 2 | **Circular Dependency** | Nœud d'ADN — topoisomérase needed | SCC with > 1 node across module boundaries | Critical |
| 3 | **Shotgun Surgery** | Couper un brin casse 10 autres | One change requires modifying > 3 files/modules | High |
| 4 | **Feature Envy** | Fonction qui appelle plus l'extérieur que son module | out-degree to other modules > out-degree within module | High |
| 5 | **Dead Weight** | Cellule morte non éliminée (pas d'apoptose) | in_degree = 0, not entry point, last modified > 3 months | Warning |
| 6 | **Hub and Spoke** | Étoile de mer sans redondance — hub tombe, tout tombe | One module has > 50% of all inter-module edges | High |
| 7 | **Layering Violation** | Courant qui remonte (court-circuit) | Lower layer calls upper layer (e.g., data layer calls API layer) | Critical |
| 8 | **Copy-Paste Cluster** | Clones cellulaires — même code, divergence silencieuse | Multiple functions with near-identical call patterns (same out-edges) | Warning |

---

## Detection heuristics

### 1. God Function
```
in_degree > P90 of all functions
AND out_degree > P75 of all functions
AND betweenness_centrality > P90
AND NOT entry_point
```
**Recommendation:** Type I refactor — extract sub-functions by responsibility.

### 2. Circular Dependency
```
Tarjan SCC with > 1 node
AND nodes span > 1 module
```
**Recommendation:** Type II refactor — introduce trait/interface at the weakest edge.

### 3. Shotgun Surgery
```
For a function F, count how many different modules contain functions
that call F or are called by F.
If module_count > 3 → shotgun surgery risk.
```
**Recommendation:** Consolidate related logic. If F is a utility, it should be in a shared module.

### 4. Feature Envy
```
For function F in module M:
external_calls = calls to functions outside M
internal_calls = calls to functions inside M
If external_calls > internal_calls × 2 → feature envy
```
**Recommendation:** Move F to the module it calls most.

### 5. Dead Weight
```
in_degree == 0
AND NOT (main | test | pub API | FFI | trait impl | macro-referenced)
AND last_git_modification > 90 days
```
**Recommendation:** Confirm with mutation testing, then remove.

### 6. Hub and Spoke
```
For each module M:
inter_module_edges(M) / total_inter_module_edges > 0.5
```
**Recommendation:** Split the hub. Introduce intermediate modules or traits.

### 7. Layering Violation
```
Define layer ordering: presentation → business → data → infra
For each call edge A→B:
If layer(A) is lower than layer(B) → violation
```
**Recommendation:** Invert the dependency (callback, trait, event).

**Layer detection heuristic:**
- Files in `api/`, `handler/`, `route/` → presentation
- Files in `service/`, `domain/`, `core/` → business
- Files in `repo/`, `db/`, `store/` → data
- Files in `infra/`, `config/`, `util/` → infra

### 8. Copy-Paste Cluster
```
For functions F1, F2:
If out_edges(F1) ∩ out_edges(F2) > 0.8 × max(|out_edges(F1)|, |out_edges(F2)|)
AND F1 and F2 are in different locations
→ probable copy-paste
```
**Recommendation:** Extract shared logic to a common function.

---

## Pre-mortem: What can go wrong with this analysis

| Risk | Symptom | Mitigation |
|------|---------|-----------|
| God score threshold too low | Too many false positives, good dispatcher functions flagged | Exclude entry points and routers explicitly |
| Cycle detection misses dynamic dispatch | `dyn Trait` calls are invisible in static graph | Flag modules using `dyn` for manual review |
| Dead code detection hits macro-generated code | Removing it breaks the build | Always confirm with mutation testing before recommending removal |
| Fiedler analysis on disconnected graph | Meaningless partitioning | Check connectivity first, analyze components separately |
| Layer detection by filename | Files named inconsistently | Fall back to import analysis if heuristic fails |
