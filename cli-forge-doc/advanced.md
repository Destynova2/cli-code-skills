# Doc Forge Advanced — Methods for approaching 100% (and beyond)

## The "120%" philosophy

100% = all necessary information is documented.
120% = the documentation **adds value** beyond the raw information: it anticipates questions, connects concepts, and speeds up comprehension.

The delta between 100% and 120% is:
- Examples that cover edge cases, not just the happy path
- ADRs that document the **rejected** alternatives (not just the final choice)
- Links between concepts that the reader would not have connected on their own
- Warnings about pitfalls that only an expert knows

---

## Part 1: The Concept Dependency Graph

### Principle

A project is not a flat list of items to document. It is a **directed graph** of concepts where each concept depends on other concepts to be understood.

```
Concept A ──→ Concept B ──→ Concept D
    │                           ↑
    └──→ Concept C ─────────────┘
```

If you document D without documenting B and C, the reader is lost. If you document B without A, same thing.

### Building the graph

**Step 1: extract the concepts**

For each source file, extract:
- Public types (structs, enums, traits, interfaces, classes)
- Public functions
- Configuration constants
- Modules/packages

Each item = one **node** in the graph.

**Step 2: extract the dependencies**

For each node, identify:
- Which types it uses as parameters or return → **direct dependency**
- Which concepts you must understand to use it → **cognitive dependency**
- Which modules it imports → **structural dependency**

Each dependency = one **edge** in the graph.

**Step 3: compute the graph metrics**

```
Node coverage = Documented nodes / Total nodes

Edge coverage = Documented links (cross-refs) / Existing links

Weighted coverage = Σ(doc_score(n) × importance(n)) / Σ importance(n)
```

Where `importance(n)` is computed as:

```
importance(n) = in_degree(n) + out_degree(n) + is_public(n) × 3

Intuition: a type used everywhere (high in_degree) is more
important to document than an internal helper used once.
```

### Gap analysis

```
gap_score(n) = importance(n) × (1 - doc_score(n))
```

Sort by decreasing `gap_score` → the items at the top are the most urgent to document. This is the **documentation priority queue**.

### Orphan detection

An **orphan** is a documented concept whose dependencies are not:

```
orphan(n) = doc_score(n) > 0.5 AND
            ∃ dep ∈ dependencies(n) : doc_score(dep) < 0.25
```

An orphan creates a **cognitive cliff**: the reader understands the node but not its prerequisites.

### Path coverage

```
A "learning path" P is an ordered sequence:
P = [concept_1, concept_2, ..., concept_k]

where each concept_i+1 depends on concept_i.

Path coverage = Fully documented paths / Total paths
```

A path is "fully documented" if ALL its nodes have `doc_score ≥ 0.75`.

**Goal**: Path coverage ≥ 95% on paths starting from the public entry points.

---

## Part 2: Documentation Entropy (Documentation Need Score)

### Principle

Shannon entropy measures uncertainty. Applied to docs:
- **High entropy** = the reader cannot predict the behavior → documentation needed
- **Low entropy** = the behavior is obvious from the name/type → no need to document

### Formula: Documentation Need Score (DNS)

```
DNS(item) = H(behavior) - I(name, types, context)

Where:
  H(behavior) = entropy of the item's behavior
                (how many possible behaviors?)
  I(name, types, context) = mutual information between
                the name/signature and the actual behavior
```

**Intuition**: if the name `get_user_by_id(id: UserId) -> Option<User>` already says everything about the behavior, the DNS is low. If `process(data: &[u8]) -> Result<Vec<Output>>` is ambiguous, the DNS is high.

### Practical heuristic (without formal computation)

| Signal | DNS | Action |
|--------|-----|--------|
| Descriptive name + strong types + no side effects | Low | Summary line is enough |
| Generic name (`process`, `handle`, `run`) | High | Full doc with examples |
| Returns `Result` or `Option` | +1 | Document the error cases |
| Takes `&[u8]`, `Any`, `impl Trait` | +1 | Document the invariants |
| Has side effects (IO, mutation, network) | +2 | Document the full behavior |
| Is `unsafe` | +3 | SAFETY documentation required |
| Has more than 3 parameters | +1 | Document the interactions between params |

### Module entropy score

```
Module_entropy = Σ DNS(item_i) for item_i ∈ module

Module_doc_quality = Σ min(DNS(item_i), doc_depth(item_i)) / Module_entropy
```

A module with `Module_doc_quality ≥ 1.0` is documented in proportion to its complexity. That is "100%".

A module with `Module_doc_quality ≥ 1.2` exceeds expectations — this is "120%":
extra examples, documented edge cases, links to the architecture, warnings on pitfalls.

---

## Part 3: Systematic error elimination

### Mutation testing for documentation

Inspired by mutation testing in software: we "break" the doc and check whether anyone would notice.

| Mutation | Test | If it passes = problem |
|----------|------|------------------------|
| Remove a paragraph | Can the reader still accomplish the task? | The doc was redundant (OK) or the test is bad |
| Invert a parameter | Will the reader be misled? | If not → the doc clarifies nothing useful |
| Change a return type in the doc | Will the reader notice? | If not → the doc is not read/useful |
| Remove an example | Is the concept still clear? | If yes → the example added nothing |

### Code / Doc cross-validation

**For each documented public item:**

```
1. Parse the doc comment → extract: params, return type, errors, panics
2. Parse the code → extract: actual signature, ? operators, panic! calls, unsafe blocks
3. Compare:
   - Params in the doc ⊆ params in the code? (completeness)
   - Params in the code ⊆ params in the doc? (exhaustiveness)
   - Documented errors ⊇ returned error types? (error coverage)
   - If unsafe in the code → SAFETY in the doc? (safety)
```

**Coherence formula:**

```
coherence(item) = |documented_facts ∩ actual_facts| / |documented_facts ∪ actual_facts|

Where:
  documented_facts = {params, returns, errors, panics, safety} extracted from the doc
  actual_facts = {params, returns, errors, panics, safety} extracted from the code

coherence = 1.0 → doc and code are in perfect sync
coherence < 0.8 → flag for review
coherence < 0.5 → critical: the doc is probably stale
```

### Automatic linting (CI integration)

| Language | Tool | What it catches |
|----------|------|-----------------|
| Rust | `clippy` (missing_docs, missing_errors_doc) | Coverage + sections |
| Rust | `cargo doc --check` | Broken links, non-compiling examples |
| Rust | `doc-coverage` (crate) | Exact coverage % |
| TypeScript | `typedoc` + plugins | Coverage + format |
| Python | `interrogate` | Docstring coverage % |
| Python | `darglint` | Param doc / signature coherence |
| Go | `golint` / `revive` | Exports without docs |
| Any | `vale` | Style, consistency, jargon |
| Any | `cspell` | Typos in the docs |

---

## Part 4: Extended DCI Score (DCI+)

### Beyond the base DCI

The base DCI measures "does it exist?". DCI+ measures "is it **good**?"

```
DCI+ = DCI_base × Quality_multiplier × Coverage_multiplier

Quality_multiplier = (1 + bonus_examples + bonus_crossrefs + bonus_adrs) / 4

  bonus_examples = 0.0 to 1.0 (proportion of items with tested examples)
  bonus_crossrefs = 0.0 to 1.0 (proportion of working intra-doc refs)
  bonus_adrs = 0.0 to 1.0 (proportion of documented architectural decisions)

Coverage_multiplier = path_coverage × node_coverage

  path_coverage = fully documented learning paths
  node_coverage = documented concept-graph nodes
```

### DCI+ grid

| DCI+ | Grade | Meaning |
|------|-------|---------|
| > 12.0 | S | Exceptional — the docs are a competitive advantage (the "120%") |
| 10.0–12.0 | A+ | Exemplary — nothing to add |
| 9.0–10.0 | A | Excellent — a few bonuses missing |
| 7.0–8.9 | B | Solid — the minimum is covered |
| < 7.0 | C or below | See base DCI |

### How to reach S (> 12.0)

Bonus checklist:

- [ ] 100% of public items have a non-tautological doc comment
- [ ] 100% of Result-returning fns have `# Errors`
- [ ] 95%+ of public items have at least one example
- [ ] All examples compile/pass
- [ ] All mentioned types are linked (intra-doc links)
- [ ] Architecture documented with at least 3 ADRs
- [ ] Getting-started tested by a non-expert in < 10 min
- [ ] llms.txt + AGENTS.md up to date
- [ ] Glossary of domain terms
- [ ] Changelog up to date
- [ ] Zero stale docs (coherence ≥ 0.95)
- [ ] Narrative transitions between all pages
- [ ] CI lint (vale + language-specific linter) passes
