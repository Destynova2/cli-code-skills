# Anti-Pattern Detection Rules

> **When to read:** During Step 4 when scanning for anti-patterns after scoring dimensions.

---

## 9 Anti-Patterns

Flag these known anti-patterns with severity and evidence:

| Anti-Pattern | Source | Detection | Severity |
|--------------|--------|-----------|----------|
| Ice Cream Cone | Codepipes | Count tests per level, check ratio — more E2E than unit = flagged | Critical |
| Cassandre Syndrome | La Taverne | Tests exist but results are never analyzed (no reporting/gating in CI) | Critical |
| Flaky Acceptance | Codepipes #6 | Retry logic in tests, `@retry`, `flaky` markers, `sleep` in test code | Warning |
| Testing Implementation | Codepipes #4 | Tests break on refactor (coupled to internals, mock overuse) | Warning |
| 100% Coverage Obsession | Codepipes #5 | Coverage targets without quality checks on what's covered | Warning |
| Late Testing | ISTQB shift-left | Tests only at E2E level, no early feedback, no unit tests | Critical |
| Smoke-less Builds | La Taverne | No smoke test suite, CI runs full suite on every commit | Warning |
| Combinatorial Explosion | La Taverne (Pairwise) | N parameters x M values tested exhaustively instead of pairwise | Info |
| Test Data Leakage | TMMi | Hardcoded secrets, shared mutable state between tests | Warning |

## Detection Heuristics

### Ice Cream Cone
- Count test files in `unit/`, `integration/`, `e2e/` directories
- If E2E count > unit count, flag as Ice Cream Cone
- Also check for manual test plans without corresponding automated tests

### Cassandre Syndrome
- Check CI config for test result publishing
- Look for quality gates (PR blocked on failure)
- If tests run but no one acts on results → Cassandre

### Flaky Acceptance
- Search for `@retry`, `@flaky`, `retry:`, `retries:` in test files
- Search for `sleep`, `wait`, `setTimeout` in test code (not helpers)
- Check CI logs for intermittent failures if accessible

### Testing Implementation (Coupling)
- High mock count relative to test count
- Tests that reference private/internal APIs
- Tests that break when code is refactored without behavior change

### 100% Coverage Obsession
- Coverage threshold set to 100% in CI config
- Trivial tests (testing getters/setters, testing framework code)
- Coverage metric without mutation testing or assertion quality checks

### Late Testing
- No unit test framework in dependencies
- All tests are E2E or manual
- Tests added only after bugs are found (reactive, not proactive)

### Smoke-less Builds
- No `smoke` tag/marker/directory in test suite
- CI runs full test suite on every push (no fast feedback loop)
- No tiered test execution strategy

### Combinatorial Explosion
- Large test matrices without pairwise reduction
- Exhaustive parameter combinations in test names
- No PICT or similar combinatorial tool in project

### Test Data Leakage
- Hardcoded passwords, tokens, or API keys in test files
- Tests sharing global state (database, files) without cleanup
- Test order dependencies (test B fails if test A doesn't run first)

## Recommendation Templates

| Anti-Pattern | Recommendation |
|--------------|---------------|
| Ice Cream Cone | Add unit tests for core logic. Target ratio: 70% unit, 20% integration, 10% E2E |
| Cassandre Syndrome | Add quality gates in CI. Block PRs on test failure. Publish test reports |
| Flaky Acceptance | Quarantine flaky tests. Fix root cause (usually timing). Use deterministic waits |
| Testing Implementation | Test behavior, not implementation. Reduce mocks. Use contract tests |
| 100% Coverage Obsession | Add mutation testing. Focus coverage on critical paths, not vanity metrics |
| Late Testing | Shift left: add unit tests during development. TDD for new features |
| Smoke-less Builds | Define a smoke suite (critical paths only). Run smoke on every commit, full suite on PR |
| Combinatorial Explosion | Adopt pairwise testing (PICT tool). Reduces test count by 80%+ with same defect detection |
| Test Data Leakage | Use test fixtures with cleanup. Generate data per test. Never hardcode secrets |
