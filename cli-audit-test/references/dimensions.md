# 12 Audit Dimensions — Detailed Scoring Criteria

> **When to read:** During Step 3 when scoring each dimension. Use the detailed criteria and evidence patterns below to assign scores 0-4.

---

## Scoring Scale (all dimensions)

| Score | Label | Meaning |
|-------|-------|---------|
| 0 | Absent | Dimension not addressed at all |
| 1 | Initial | Mentioned but vague, no concrete approach |
| 2 | Basic | Present but incomplete, missing key elements |
| 3 | Good | Well-defined, covers most best practices |
| 4 | Excellent | Comprehensive, specific techniques, measurable criteria, CI-integrated |

---

## D1 — Scope & Objectives (8%)

**Source:** ISTQB Foundation v4.0 §5, TMMi Level 2

Check for:
- Test objectives explicitly stated and aligned with business/technical goals
- Scope clearly defined (what IS tested, what is NOT, and why)
- Exclusions documented with rationale
- Test levels identified (composant, intégration, système, intégration système, acceptation — ISTQB 2023)
- Relationship between test plan and higher-level test strategy/policy

**Scoring:**
- 0: No scope or objectives anywhere
- 1: Vague scope ("we test the API")
- 2: Scope exists but no exclusions or rationale
- 3: Clear scope with exclusions and level mapping
- 4: Scope aligned to risk analysis, traceability to requirements, explicit rationale for every exclusion

---

## D2 — Test Design Techniques (12%)

**Source:** ISTQB Foundation v4.0 §4, La Taverne du Testeur (Pairwise, BVA articles)

Check for explicit use of recognized techniques:

| Technique | What it covers | Where to look |
|-----------|---------------|---------------|
| Equivalence Partitioning (EP) | Input classes | Test case names, comments, directories |
| Boundary Value Analysis (BVA) | Edge values | Test names with "limit", "boundary", "max", "min" |
| Decision Table | Business rule combinations | Files named `*decision*`, `*policy*`, truth tables in comments |
| State Transition | Stateful workflows | Files named `*state*`, `*circuit*`, `*transition*`, state diagrams |
| Pairwise / Combinatorial | Parameter interactions | PICT files, `*.pict`, pairwise directories, combination matrices |
| Use Case / Scenario | End-to-end flows | `*happy*`, `*scenario*`, `*flow*` directories |
| Error Guessing | Experience-based | `*fuzz*`, `*edge*`, `*negative*` directories |
| Exploratory | Session-based | Charters, `*exploratory*`, SBTM references |

**Scoring:**
- 0: No techniques mentioned or identifiable
- 1: Only happy-path tests, no technique awareness
- 2: 1-2 techniques used (typically EP + scenario)
- 3: 3-4 techniques with clear mapping to features
- 4: 5+ techniques, each justified for its target (BVA for numeric boundaries, decision tables for business rules, state transition for workflows, pairwise for combinatorial explosion)

---

## D3 — Test Pyramid Balance (8%)

**Source:** Mike Cohn, Codepipes anti-patterns, La Taverne (article "Pourquoi une pyramide pour les tests ?")

Check the distribution across test levels:

| Shape | Pattern | Verdict |
|-------|---------|---------|
| Pyramid | Unit > Integration > E2E | Healthy |
| Diamond | Few unit, many integration, few E2E | Acceptable if justified |
| Ice Cream Cone | Few unit, some integration, many manual/E2E | Anti-pattern |
| Cupcake | Bloated at every level | Wasteful |
| Inverted Pyramid | Mostly E2E/manual, no unit | Critical anti-pattern |

**How to assess:**
- Count test files per level/directory
- Check for unit test framework (cargo test, jest, pytest, etc.)
- Check for integration test setup (test containers, mocks, stubs)
- Check for E2E framework (hurl, playwright, cypress, etc.)

**Scoring:**
- 0: Only one level exists (typically E2E or nothing)
- 1: Two levels but inverted (more E2E than unit)
- 2: Two levels in correct ratio
- 3: Three levels with reasonable ratio
- 4: Full pyramid with quantified targets, smoke tests identified, and clear rationale for ratio

---

## D4 — Coverage & Traceability (12%)

**Source:** ISTQB v4.0 §5.3, TMMi Level 2-3

Check for:
- Requirements Traceability Matrix (RTM) or equivalent mapping
- Coverage criteria defined and measurable (code coverage %, feature coverage, risk coverage)
- KPIs tied to quality gates
- Each requirement/feature has at least one test
- Coverage gaps explicitly acknowledged

**Scoring:**
- 0: No traceability, no coverage metrics
- 1: Informal "we test everything important"
- 2: Feature list exists but no formal mapping to tests
- 3: RTM or structured mapping, coverage targets defined
- 4: Full RTM with bidirectional traceability, coverage measured in CI, gaps documented with risk acceptance

---

## D5 — Negative Testing & Security (8%)

**Source:** OWASP, La Taverne ("invariants : ce qui ne doit PAS arriver"), Codepipes anti-pattern #12

Check for:
- Invalid input tests (malformed data, wrong types, empty, oversized)
- Error handling path tests
- Boundary violations (not just boundaries — exceeding them)
- Security invariants (token forgery, injection, escalation, information leakage)
- Dedicated negative/security test directory or naming pattern

**What to look for in test names/structure:**
- `*negative*`, `*invalid*`, `*malicious*`, `*inject*`, `*forge*`, `*tamper*`
- `*401*`, `*403*`, `*413*`, `*brute*`, `*replay*`, `*traversal*`
- `*fuzz*`, `*overflow*`, `*crlf*`

**Scoring:**
- 0: Only happy paths tested
- 1: A few error cases but no systematic approach
- 2: Some negative tests exist but no security invariants
- 3: Dedicated negative suite, covers main attack vectors
- 4: Comprehensive negative suite with named invariants (e.g., N0-N11), security techniques (OWASP top 10), and separation from functional tests

---

## D6 — Non-Functional Testing (10%)

**Source:** ISO 25010, ISTQB v4.0 §2.2, TMMi Level 3

Check for coverage of NFR categories:

| NFR | Evidence |
|-----|----------|
| Performance / Load | `*load*`, `*perf*`, `*stress*`, k6/gatling/locust files |
| Security | `*secu*`, OWASP scans, SAST/DAST config |
| Accessibility | `*a11y*`, WCAG checks, axe config |
| Reliability | Chaos testing, `*failover*`, toxiproxy, circuit breaker tests |
| Compatibility | Multi-browser/device matrices |
| Observability | Audit log tests, metrics verification |

**Scoring:**
- 0: No NFR tests at all
- 1: One NFR category addressed informally
- 2: 2-3 NFR categories with dedicated tests
- 3: 4+ NFR categories, tools configured
- 4: Comprehensive NFR strategy with targets (SLOs), tools, and CI integration

---

## D7 — Risk Analysis & Prioritization (8%)

**Source:** ISTQB v4.0 §5.2, TMMi Level 2

Check for:
- Risk identification (probability × impact)
- Risk-based test prioritization (critical features get more test effort)
- Mitigation plans for identified risks
- Test order reflecting risk priority
- Contingency plans for blocked/delayed testing

**Scoring:**
- 0: No risk analysis
- 1: Informal priority ("we test important stuff first")
- 2: Risk list exists but not formally linked to test effort
- 3: Risk matrix with probability/impact, test priority derived from it
- 4: Quantified risk model, dynamic re-prioritization, defect clustering analysis

---

## D8 — Automation Strategy (10%)

**Source:** La Taverne (Smoke tests article), TMMi Level 3, Codepipes anti-patterns

Check for:
- What is automated vs. manual (and why)
- Automation framework selection with rationale
- Test data management strategy
- Maintenance plan for automated tests (flaky test policy, refactoring)
- Smoke test suite identified and automated

**Scoring:**
- 0: No automation
- 1: Some tests automated but no strategy (ad-hoc)
- 2: Automation exists with tool choice, but no maintenance/data plan
- 3: Clear automation strategy with framework, smoke suite, data management
- 4: Mature strategy with flaky test handling, parallel execution, test isolation, deterministic data, and clear manual/auto split rationale

---

## D9 — CI/CD Integration (8%)

**Source:** Shift-left testing, TMMi Level 3-4

Check for:
- Tests run automatically on commit/PR
- CI pipeline defined with test jobs
- Feedback time acceptable (< 15min for unit/integration, < 1h for E2E)
- Quality gates (PR blocked if tests fail)
- Test results published (artifacts, reports)
- Environment setup automated (up/down scripts)

**Scoring:**
- 0: No CI integration
- 1: CI exists but tests not integrated
- 2: Some tests in CI but no quality gates
- 3: Full test suite in CI with gates, reasonable feedback time
- 4: Multi-stage pipeline (unit → integration → E2E), parallel execution, environment auto-provisioning, test artifacts published

---

## D10 — Entry/Exit Criteria (6%)

**Source:** ISTQB v4.0 §5.1, TMMi Level 2

Check for:
- Entry criteria: preconditions before testing starts (env ready, build passes, test data available)
- Exit criteria: when testing is "done" (coverage thresholds, defect density limits, no critical bugs)
- Quality gates at each test level transition
- DoD (Definition of Done) includes testing criteria

**Scoring:**
- 0: No criteria defined
- 1: Vague ("testing is done when we've tested enough")
- 2: Exit criteria exist but not measurable
- 3: Measurable entry + exit criteria
- 4: Quantified criteria with thresholds, automated verification in CI, per-level gates

---

## D11 — Exploratory & Human Testing (5%)

**Source:** ISTQB v4.0 §4.4, Session-Based Test Management (SBTM)

Check for:
- Exploratory testing sessions planned (not just "someone will poke around")
- Charters defined (focused exploration areas)
- Findings fed back into the formal test suite
- Time-boxed sessions with debriefs
- Balance between scripted and exploratory

**Scoring:**
- 0: No exploratory testing mentioned
- 1: "We'll do some manual testing" without structure
- 2: Exploratory testing acknowledged, some ad-hoc sessions
- 3: Structured charters, time-boxed sessions, findings documented
- 4: Full SBTM with charters, debriefs, regression integration, and metrics

---

## D12 — Environment & Test Data (5%)

**Source:** TMMi Level 2, ISTQB v4.0 §5.1

Check for:
- Test environments specified (local, CI, staging, production-like)
- Environment setup automated (scripts, containers, IaC)
- Test data strategy (generation, anonymization, cleanup)
- Mock/stub strategy for external dependencies
- Environment parity with production

**Scoring:**
- 0: No environment or data strategy
- 1: "We test locally" without setup documentation
- 2: Environment described, some setup scripts
- 3: Automated environment setup, mock strategy, data generation
- 4: Full environment-as-code, deterministic test data, mocks for all external deps, cleanup on teardown, prod parity documented
