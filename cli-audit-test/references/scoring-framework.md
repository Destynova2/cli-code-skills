# Scoring Framework Reference — cli-audit-test

## Weight Rationale

| Dimension | Weight | Why this weight |
|-----------|--------|----------------|
| D2 — Test Design Techniques | 12% | Core differentiator between amateur and professional testing. A plan without techniques is just a wish list |
| D4 — Coverage & Traceability | 11% | Without traceability, you can't answer "what's tested?" or "what's the blast radius of this change?" |
| D6 — Non-Functional Testing | 9% | NFR bugs are the most expensive to fix late. Early NFR testing has highest ROI |
| D8 — Automation Strategy | 9% | Automation without strategy creates maintenance debt. Strategy without automation wastes time |
| D13 — Semantic Drift Detection | 8% | Silent behavioral changes are the most dangerous bugs — they pass all tests, compile fine, and ship to prod unnoticed |
| D1 — Scope & Objectives | 7% | Foundation — everything else builds on clear scope |
| D3 — Pyramid Balance | 7% | Shape determines feedback speed and maintenance cost |
| D5 — Negative Testing | 8% | "Ce qui ne doit PAS arriver" is often more critical than "ce qui doit arriver" |
| D7 — Risk Analysis | 7% | Testing everything equally is waste. Risk-based prioritization is essential |
| D9 — CI/CD Integration | 7% | Tests not in CI are tests that will be forgotten |
| D10 — Entry/Exit Criteria | 5% | Quality gates prevent premature releases |
| D11 — Exploratory Testing | 5% | Complements scripted tests. Less weight because it's inherently less auditable |
| D12 — Environment & Data | 5% | Enabler dimension — bad env/data undermines all other testing |

## TMMi Level Mapping

### Level 2 — Managed
Required:
- D1 ≥ 2 (Scope defined per project)
- D4 ≥ 2 (Basic coverage tracking)
- D7 ≥ 1 (Some risk awareness)
- D10 ≥ 2 (Entry/exit criteria exist)
- D12 ≥ 2 (Environment documented)

### Level 3 — Defined
Level 2 + :
- D2 ≥ 3 (Multiple techniques used systematically)
- D6 ≥ 2 (NFR testing addressed)
- D8 ≥ 2 (Automation strategy exists)
- D11 ≥ 2 (Exploratory testing structured)
- Organization-wide test process (not just per-project)

### Level 4 — Measured
Level 3 + :
- D4 ≥ 4 (Quantitative coverage with CI measurement)
- D9 ≥ 3 (Full CI integration with quality gates)
- D10 ≥ 3 (Measurable, enforced criteria)
- Statistical process control (defect trends, prediction)

### Level 5 — Optimization
Level 4 + :
- All dimensions ≥ 3
- D2 ≥ 4, D4 ≥ 4, D8 ≥ 4
- D13 ≥ 3 (At least 2 drift detection layers with CI integration)
- Continuous improvement process (retrospectives on test strategy)
- Defect prevention (root cause analysis feeding test design)
- Innovation (mutation testing, property-based testing, contract testing, AI-assisted)

## Technique Detection Heuristics

### Pairwise / Combinatorial
- **Files**: `*.pict`, `*pairwise*`, `*combinatorial*`
- **Content**: PICT model syntax (`parameter: value1, value2`), `IF [x] THEN [y]` constraints
- **Structure**: Large number of generated test cases from a model file
- **Signal**: Comment mentioning "reduces N combinations to M cases"

### Boundary Value Analysis (BVA)
- **Pattern**: Tests at exact boundary, boundary-1, boundary+1
- **Names**: `*limit*`, `*boundary*`, `*threshold*`, `*max*`, `*min*`, `*edge*`
- **Content**: Sequential numeric values near a limit (e.g., 79%, 80%, 100%, 101%)
- **Signal**: Tests with values like `MAX_SIZE`, `MAX_SIZE + 1`, `MAX_SIZE - 1`

### Decision Table
- **Files**: `*decision*`, `*policy*`, `*rule*`, `*matrix*`
- **Content**: Truth-table-like structures, multiple condition combinations
- **Structure**: Tests organized as condition1 × condition2 × ... → expected outcome
- **Signal**: Comments with tables showing input combinations and expected results

### State Transition
- **Files**: `*state*`, `*circuit*`, `*transition*`, `*lifecycle*`, `*flow*`
- **Content**: Tests following a sequence (state1 → event → state2)
- **Structure**: Tests named with state names (e.g., `closed_to_open`, `halfopen_probe_fail`)
- **Signal**: Setup that configures initial state before testing transitions

### Equivalence Partitioning (EP)
- **Pattern**: One test per class of inputs (valid, invalid, special)
- **Names**: `*valid*`, `*invalid*`, `*class*`, `*partition*`
- **Content**: Representative values from different input domains
- **Signal**: Comments mentioning "representative", "partition", "class"

### Error Guessing / Fuzz
- **Files**: `*fuzz*`, `*edge*`, `*chaos*`, `*malformed*`
- **Content**: Unusual inputs (empty, null, huge, special chars, unicode)
- **Signal**: Tests that feel like "what would a malicious user try?"

## Anti-Pattern Detection Rules

### Ice Cream Cone
**Detection**: `count(E2E tests) > count(unit tests)`
**Severity**: Critical if ratio > 2:1, Warning if ratio > 1:1
**Recommendation**: Add unit tests for business logic before adding more E2E

### Cassandre Syndrome
**Detection**: Tests exist but:
- No CI integration (tests never run automatically)
- CI exists but no quality gates (failures don't block)
- Reports generated but no action taken (stale issues)
**Severity**: Critical — tests that don't block are tests that don't matter
**Recommendation**: Add CI quality gates, block merges on test failure

### Smoke-less Builds
**Detection**: No identified smoke test suite; CI runs the full suite on every commit
**Severity**: Warning
**Recommendation**: Identify a fast subset (< 2min) as smoke tests, run on every commit. Full suite on PR/nightly.

### Combinatorial Explosion
**Detection**: Large test count without pairwise/PICT, many similar test files differing only in parameter combinations
**Severity**: Warning
**Recommendation**: Introduce PICT model to reduce N×M exhaustive combinations to ~50 pairwise cases

### Test Data Leakage
**Detection**: Hardcoded passwords, API keys, or secrets in test files. Shared mutable state (database) without cleanup.
**Severity**: Critical for secrets, Warning for shared state
**Recommendation**: Use generated test materials, cleanup scripts, and `.gitignore` for sensitive files

### Silent Drift Blindness
**Detection**: No mutation testing, no contract/snapshot testing, no property-based invariants. All tests are pure example-based (fixed input → fixed expected output).
**Severity**: Critical — the most dangerous bugs (silent behavioral drift) go completely undetected
**Recommendation**: Start with the layer matching your biggest risk: `cargo-mutants` for logic drift, `insta`/`Pact` for API contract drift, `proptest` for domain invariant drift

## Comparative Benchmarks

Based on industry surveys and testing maturity assessments:

| Project Type | Typical Score | Target Score |
|-------------|--------------|-------------|
| Startup MVP | 15-30 | 40+ |
| Enterprise internal tool | 30-50 | 60+ |
| SaaS product | 40-60 | 75+ |
| Financial / Healthcare | 50-70 | 85+ |
| Safety-critical (aviation, medical) | 70-85 | 90+ |
| Open-source library (mature) | 40-65 | 70+ |

## Sources

### Books
- Cem Kaner, James Bach, Bret Pettichord — "Lessons Learned in Software Testing"
- Lisa Crispin, Janet Gregory — "Agile Testing" + "More Agile Testing"
- Gerald Weinberg — "Perfect Software and Other Illusions about Testing"
- Robert C. Martin — "Clean Code" (test chapter)
- John Ousterhout — "A Philosophy of Software Design"

### Standards & Frameworks
- ISTQB Foundation Level Syllabus v4.0 (2023)
- TMMi Framework R1.2 (tmmi.org)
- ISO 25010:2023 — Quality characteristics
- ISO 29119 — Software Testing Standard

### Online
- Codepipes — "Software Testing Anti-patterns" (blog.codepipes.com)
- Alister Scott — "Testing Pyramids & Ice-Cream Cones"
- Ministry of Testing — "Community Guide to Test Plans"
- La Taverne du Testeur (latavernedutesteur.fr):
  - "Les niveaux de test — 2023" (ISTQB 5 levels)
  - "Les smoke tests" (smoke test strategy)
  - "Pourquoi une pyramide pour les tests ?" (pyramid analysis)
  - "Le processus de test" (ISTQB test process)
  - "Ce que nous apprend la mythologie : Cassandre" (Cassandre syndrome)
  - "Techniques basées sur les spécifications — Pairwise"

### Podcasts
- TestGuild Automation Podcast (Joe Colantonio)
- AB Testing — agile testing practices
- The Evil Tester Show — advanced testing philosophy
- Quality Sense (Abstracta) — testing maturity

### Tools Referenced
- PICT (Microsoft) — pairwise test case generation
- Hurl — HTTP testing
- Toxiproxy — chaos/fault injection
- k6/Gatling/Locust — load testing
- SonarQube — static analysis (the "Cassandre" tool per La Taverne)
- cargo-mutants / mutmut / Stryker / pitest — mutation testing (D13 MMR)
- Pact / consumer-driven contracts — contract testing (D13 CRISPR)
- insta / jest snapshots / approval-tests — snapshot testing (D13 CRISPR)
- proptest / quickcheck / Hypothesis / fast-check — property-based testing (D13 Epigenetics)
