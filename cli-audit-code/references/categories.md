# Audit Categories — Detailed Checks

> **When to read:** Step 3 (Score all dimensions). Contains the full check lists per category.

---

## C1: NAMING & READABILITY (8%)

**Source:** Robert C. Martin (Clean Code), Google Style Guides

Check for:
- Single-letter variables (except `i/j/k` in tight loops, `_` ignores)
- Cryptic abbreviations (`cfg`, `mgr`, `ctx` are OK if universal; `h`, `p`, `s` are not)
- Names that don't reveal intent
- Misleading names (name suggests different behavior than actual)
- Hungarian notation or type-encoded names
- Non-searchable names
- Magic numbers without named constants (hardcoded `1024`, `0.01`, `3` in conditionals)

## C2: FUNCTIONS & COGNITIVE COMPLEXITY (12%)

**Source:** Martin, Ousterhout, SonarSource Cognitive Complexity (2017)

Check for:
- Functions longer than **30 lines** (flag), longer than **60 lines** (critical)
- Functions doing more than one thing (test: can you extract a meaningful sub-function?)
- More than **3 parameters** (flag), more than **5** (critical)
- Boolean flag parameters that select behavior (should be split)
- Side effects hidden in getter/query functions
- Functions at wrong abstraction level (mixing high-level orchestration with low-level details)
- **Cognitive complexity > 15** (count: +1 per branch/loop, +1 extra per nesting level)

**Cognitive Complexity formula (SonarSource):**
- +1 per `if`, `else if`, `else`, `for`, `while`, `match` arm, `&&`, `||`
- +1 extra per nesting level (the key differentiator from cyclomatic)
- Target: < 15 per function

## C3: MODULE DESIGN — DEEP VS SHALLOW (10%)

**Source:** John Ousterhout (A Philosophy of Software Design)

Check for:
- **Shallow modules**: complex interface, trivial implementation
- **Pass-through methods**: layers that add nothing, just forward calls
- Files longer than **500 lines** (flag), **1000 lines** (critical)
- God modules doing too many things
- Related code not vertically close
- Inconsistent ordering (public API scattered among private helpers)
- **Information leakage**: implementation details exposed across module boundaries
- Circular or tangled dependencies between modules

## C4: DRY & CHANGE AMPLIFICATION (8%)

**Source:** Martin (DRY), Ousterhout (Change Amplification), Fowler (Refactoring)

Check for:
- Copy-pasted code blocks (>5 similar lines appearing 2+ times)
- Missing shared abstractions for repeated patterns
- Over-abstraction (premature generalization for single-use patterns)
- **Change amplification**: would a small requirement change touch many files?
- **Shotgun surgery** (Fowler): one logical change requires editing many classes

## C5: ERROR HANDLING & ROBUSTNESS (10%)

**Source:** Martin, Michael Nygard (Release It!), SonarQube Reliability axis

Check for:
- `unwrap()` / `expect()` in non-test code without justification
- Swallowed errors (empty catch/match arms that silently continue)
- Error messages that don't help debugging (no context about what failed or why)
- Inconsistent error strategy (mixing `Result`, `panic!`, `Option` for similar operations)
- Missing error propagation (manual matching where `?` would suffice)
- **Parse, don't validate** (Alexis King): are inputs validated once at boundaries and turned into typed representations?

## C6: TYPE SAFETY & LANGUAGE IDIOMS (8%)

**Adapt to detected language.** For Rust:
- Unnecessary `.clone()` where a borrow would work
- `&String` / `&Vec<T>` parameters instead of `&str` / `&[T]`
- Manual loops where iterators would be clearer
- `#[allow(dead_code)]` or `#[allow(clippy::*)]` without justification
- Needless `mut` on variables
- Async functions that don't await anything
- **Make illegal states unrepresentable** (Yaron Minsky): does the type system prevent invalid combinations?

For other languages: adapt to equivalent idioms (TypeScript strict mode, Python type hints, Go error patterns, etc.)

## C7: COMMENTS & PUBLIC API DOCS (5%)

**Source:** Martin (Clean Code), RFC 1574

Check for:
- Redundant comments restating the code
- Commented-out code (should be deleted)
- TODO/FIXME/HACK without tracking (stale debt)
- Missing doc comments on **public API** items
- **Good comments** (don't flag): intent explanations, consequence warnings, legal, safety justifications

Lower weight because good code should be self-documenting. Doc-on-public-API still matters.

## C8: TEST QUALITY (12%)

**Source:** Martin, Codepipes anti-patterns, Google testing blog

Check for:
- Tests with multiple unrelated assertions (should be split)
- Test names that don't describe what's being tested
- Duplicated test setup (missing fixtures/helpers)
- No tests for error paths
- Tests coupled to implementation details (will break on refactor)
- Missing tests for public API functions
- **Flaky tests** (retry logic, sleep-based waits)
- **Ice cream cone**: more E2E tests than unit tests

## C9: SECURITY & INPUT VALIDATION (10%)

**Source:** OWASP, SonarQube Security axis, CWE

Check for:
- User input used without validation/sanitization
- SQL/command injection vectors (string concatenation with user data)
- Hardcoded secrets, passwords, API keys
- Overly permissive CORS, authentication bypasses
- Sensitive data in logs or error messages
- Missing rate limiting on public endpoints
- **Parse, don't validate at boundaries**

## C10: IMMUTABILITY & STATE MANAGEMENT (7%)

Check for:
- Mutable state where immutable would work
- Functions with hidden side effects
- `static mut` or global mutable singletons
- Large mutable structs where a builder or smaller types would be clearer
- Shared mutable state without synchronization

## C11: COGNITIVE LOAD & CONTROL FLOW (5%)

**Source:** Ousterhout, SonarSource

Check for:
- Deep nesting (>3 levels of `if/match/for`)
- Negative conditionals (`if !is_valid` instead of `if is_invalid`)
- Long boolean expressions without explanatory variables
- Complex control flow (early returns mixed with deep nesting)
- **Unknown unknowns**: non-obvious behavior not documented or named clearly

## C12: DEPENDENCIES & ARCHITECTURE (5%)

**Source:** Adam Tornhill (Software Design X-Rays), Martin (Clean Architecture)

Check for:
- Circular dependencies between modules
- High coupling: many imports from one module to another
- **Change coupling** (Tornhill): files that always change together signal hidden dependency
- Dependency on concrete implementations instead of abstractions (DIP violation)
- God file that everything imports
