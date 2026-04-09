# Parallel Exploration — Best-of-N & Benchmarking

> Sources: AlphaCodium (Qodo), AlphaCode (DeepMind), Cursor Shadow Workspace, Claude Code Competing Hypotheses, ICLR 2025 MAD analysis

## When to use this pattern

| Situation | Standard pattern (1 commis/task) | Parallel exploration (N commis/task) |
|-----------|----------------------------------|--------------------------------------|
| Well-defined task, one obvious approach | ✅ | ❌ overkill |
| Architecture choice (2-3 valid options) | ❌ anchoring risk | ✅ |
| Complex bug, unknown root cause | ❌ confirmation bias | ✅ (competing hypotheses) |
| Performance optimization (algorithm, data structure) | ❌ | ✅ (benchmark required) |
| Refactoring with multiple possible splits | ❌ | ✅ (compare the results) |

**Rule: do NOT use this for more than 30% of a sprint's tasks.** Parallel exploration costs 2-3x more tokens. Reserve it for high-impact decisions.

## Pattern 1 — Competing Hypotheses (Debug)

For complex bugs where the root cause is not obvious.

```
The Chef identifies a bug
  │
  ▼
Formulate 2-3 distinct hypotheses:
  H1: "The bug comes from an invalid cache"
  H2: "The bug comes from a race condition"
  H3: "The bug comes from incorrect parsing"
  │
  ▼
Spawn 1 commis per hypothesis (separate worktree each)
  - Each commis receives ONE hypothesis
  - Mission: prove OR refute the hypothesis
  - Produce: evidence (logs, tests, reproducing scenario)
  │
  ▼
Commis send their conclusions to the Sous-Chef
  │
  ▼
The Sous-Chef compares:
  - Which hypothesis has reproducible proof?
  - Which hypothesis explains ALL the symptoms?
  │
  ▼
The Chef validates and assigns the fix to the winning commis
```

**Anti-pattern: DO NOT have the commis debate each other.** The Sous-Chef judges, the commis never talk to each other. See "Debate vs Parallel" below for why.

## Debate vs Parallel — When different context helps (and when it doesn't)

> Source: ICLR 2025 — "Multi-LLM-Agent Debate: A Critical Assessment"

Multi-agent debate (agents arguing with each other to converge) **rarely beats a single agent + chain-of-thought**. Three reasons:

1. **Position bias** — each agent defends its initial answer instead of hunting for the truth
2. **Complacency** — the "losing" agent often concedes by default, not conviction
3. **3-5x cost** — for an equivalent or worse result

### Decision table: debate, parallel, or judge?

| The truth is... | Example | Method | Who decides | Debate between commis? |
|-----------------|---------|--------|-------------|------------------------|
| **Verifiable by a test** | "Which algo is fastest?" | Parallel + bench (Pattern 2) | The numbers | ❌ Useless — the tests speak |
| **Verifiable by CI** | "Which refactoring breaks least?" | Parallel + CI (Pattern 2) | Green/red CI | ❌ Useless — CI decides |
| **Factual but distributed** | "Where does this bug come from?" (agent A sees logs, B code, C metrics) | Competing hypotheses (Pattern 1) | The Sous-Chef synthesizes the facts | ❌ Each reports its evidence, no exchange |
| **Subjective / taste** | "How should we name this module?" | Escalate to the patron (human) | The human | ❌ Agents have no taste |
| **Normative / political** | "Which framework do we adopt?" | Escalate to the patron (human) | The human (strategic decision) | ❌ Not a technical decision |

### Different context is useful FOR collection, NOT for decision-making

```
USEFUL:                                     USELESS:
Agent A reads the server logs              Agent A argues "it's a cache bug"
Agent B reads the source code              Agent B argues "no, race condition"
Agent C reads Grafana metrics              Agent C argues "you're all wrong"
         │                                              │
         ▼                                              ▼
Sous-Chef: "A found an error in            3 rounds of debate → the most verbose
the logs at 14:32, B confirms that         agent 'wins' → confidence bias
fn parse() does not handle this case,      → potentially wrong decision
C shows a latency spike at the same
moment → root cause identified"
```

**Rule for the brigade:** Commis are **investigators**, not **lawyers**. They collect facts in their domain. The Sous-Chef is the **judge** who synthesizes. Commis never plead.

### When a real debate might work (rare case, <5%)

The only case where exchange between agents adds value:

- **Cross code review**: Commis A re-reads Commis B's code and vice versa
- But that is NOT a debate — it is a **re-read** with factual feedback ("line 42: this assertion can panic if X is empty")
- The Sous-Chef already owns this role (quality gates)
- So even this case is covered without inter-commis debate

### Summary: 3 rules

1. **If it's measurable → benchmark, not debate** (Pattern 2, comparison grid)
2. **If it's factual → collect in parallel, synthesize via the judge** (Pattern 1, Sous-Chef)
3. **If it's subjective → escalate to the human** (Appel au patron)

## Pattern 2 — Competing Approaches (Architecture / Refactoring)

For choices with 2-3 valid solutions.

```
The Chef defines the problem + constraints
  │
  ▼
Assign to 2-3 commis:
  Commis A: "Implement with approach X (e.g., trait objects)"
  Commis B: "Implement with approach Y (e.g., enum dispatch)"
  Commis C: "Implement with approach Z (e.g., generics)"
  │
  ▼
Each commis produces:
  1. A working implementation (compiles + tests pass)
  2. Benchmark (if applicable)
  3. Notes: complexity, lines of code, maintainability
  │
  ▼
The Sous-Chef runs the comparison grid (see below)
  │
  ▼
The Chef picks + the losing branches are deleted
```

### Comparison grid

The Sous-Chef fills this grid for each approach:

| Criterion | Weight | Approach A | Approach B | Approach C |
|-----------|--------|-----------|-----------|-----------|
| Tests pass | 10 | ✅/❌ | ✅/❌ | ✅/❌ |
| CQI (/cli-audit-code) | 5 | score | score | score |
| Lines of code | 3 | N | N | N |
| Cyclomatic complexity | 4 | N | N | N |
| Performance (bench) | 5 (if applicable) | ms/op | ms/op | ms/op |
| Coupling (/cli-audit-tangle) | 4 | score | score | score |
| Ease of future extension | 3 | 1-5 | 1-5 | 1-5 |

**Score** = sum(weight × normalized grade). The approach with the best score wins.

**Knock-out filter**: if tests don't pass → immediately eliminated (no score).

## Pattern 3 — AlphaCodium Flow (Iterative Generation + Selection)

For algorithmic problems or critical implementations. Lighter than full parallel (2-3 variants, not N).

```
Phase 1 — Reflection
  The commis analyzes the problem:
  - Goals, constraints, edge cases
  - Why each test expects this result
  │
Phase 2 — Generate 2-3 solutions
  The commis produces 2-3 variants IN THE SAME WORKTREE:
  - solution_a.rs, solution_b.rs, solution_c.rs
  │
Phase 3 — Rank
  The commis runs the tests on each variant
  Eliminates the failing ones
  Keeps the simplest among those that pass
  │
Phase 4 — Generate adversarial tests
  The commis invents 6-8 extra edge cases
  Runs them against the chosen solution
  │
Phase 5 — Iterate
  If an adversarial test fails → fix and return to Phase 4
  If 3 iterations without convergence → escalate to the Chef
  │
Phase 6 — Deliver
  Delete the losing variants
  Commit only the winning solution
```

**Advantage**: only costs one commis (no worktree duplication).

## Integration with the boss

### In the PERT

Mark parallel-exploration tasks:

```
┌──────────────────────────────────┐
│  T3: Refactor auth module        │
│  MODE: EXPLORATION (2 commis)    │
│  Approach A: trait objects        │
│  Approach B: enum dispatch        │
│  GATE: comparison + selection     │
└──────────────────────────────────┘
```

### In shared-state.md

Add a section:

```markdown
## Explorations in progress

| Task | Commis | Approach | Status | Score |
|------|--------|----------|--------|-------|
| T3 | commis-1 | trait objects | In progress | - |
| T3 | commis-2 | enum dispatch | In progress | - |
```

### In the Chef prompt

```
For tasks marked EXPLORATION:
1. Spawn N commis, each with a different approach
2. Wait until all are done (or 30min timeout)
3. Ask the Sous-Chef to run the comparison grid
4. Pick the winning approach
5. Delete the losing worktrees/branches
6. Continue the PERT with the winning branch
```

### When the Chef decides to use exploration

The Chef proposes parallel exploration if:
- The user explicitly asked "compare the approaches"
- The task has a scope > S in the mitosis
- There is a non-trivial architecture choice (2+ applicable patterns)
- The user did not specify a precise approach

The Chef asks for user confirmation before launching (x2-3 token cost).
