# Audit Categories — Detailed Checks

> **When to read:** Step 3 (Score all dimensions). Contains the full check lists per category.

---

## C1: NAMING & READABILITY (8%)

**Source:** Robert C. Martin (Clean Code), Google Style Guides, Microsoft Rust Guidelines (M-CONCISE-NAMES)

Check for:
- Single-letter variables (except `i/j/k` in tight loops, `_` ignores)
- Cryptic abbreviations (`cfg`, `mgr`, `ctx` are OK if universal; `h`, `p`, `s` are not)
- Names that don't reveal intent
- Misleading names (name suggests different behavior than actual)
- Hungarian notation or type-encoded names
- Non-searchable names
- Magic numbers without named constants (hardcoded `1024`, `0.01`, `3` in conditionals)
- **Weasel words** (M-CONCISE-NAMES): names that add no information and signal indecision. Flag identifiers and file names containing `Manager`, `Service`, `Factory`, `Helper`, `Util(s)`, `Common`, `Misc`, `Handler`, `Processor`, `Data`, `Info` when they're the *primary* name. Example: an item handling many bookings should be `Bookings`, not `BookingService`. Rust idiom: prefer `Builder` over `Factory`, prefer `impl Fn() -> Foo` over factory parameters. The weasel-word rule applies to **file names too** — see C3 (Catch-All Module).

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

### Extraction trade-off — when to extract vs. inline

> **Biomimetic frame — supersaturation & crystallization.** A solute stays dissolved until concentration crosses the saturation point; then it crystallizes spontaneously. Code is the same: a block stays *dissolved* (inline) until enough criteria converge, then it *crystallizes* into a function. Extracting too early (under-saturated) or too late (over-saturated) both hurt readability.

The cleanest criterion is the **call count**, because it's the only fully objective one. The rest is judgment.

Extract when **≥ 2** of these are true:

| Criterion | Threshold "extract" |
|-----------|---------------------|
| Times called | ≥ 2 (DRY — avoid divergence) |
| Block length | > ~7-10 lines |
| Different abstraction level than caller | yes |
| Independently testable / nameable | yes |

**Decision shortcuts:**
- ≥ 2 calls → **always extract** (DRY trumps inline preference — call count is the supersaturation point)
- 1 call + < 10 lines + same abstraction level → **inline** (under-saturated; extracting adds noise without nucleation)
- 1 call + isolable business logic / testable in isolation → **extract** (nameable + testable = enough nucleation surface)

**Anti-patterns to flag in BOTH directions:**
- *Long Method* (Fowler) — block should crystallize but hasn't. Covered by the >30/>60 line rule above.
- *Tiny Wrapper / Premature Extraction* — a 3-line helper called from one place at the same abstraction level. **Apoptosis check**: a helper that no longer earns its existence should be reabsorbed. Flag when a private function is called exactly once, is < 10 lines, *and* operates at the same abstraction level as its sole caller.
- *Speculative Extraction* — helper extracted "in case we need it later" with no second caller in sight. Same as above plus signals over-engineering.

Uncle Bob's underlying rule still wins ties: **a function does one thing at one level of abstraction**. If you have to mentally "zoom in" to read a block, it's a candidate to extract regardless of call count.

## C3: MODULE DESIGN — DEEP VS SHALLOW (10%)

**Source:** John Ousterhout (A Philosophy of Software Design), Microsoft Rust Guidelines (M-SMALLER-CRATES)

Check for:
- **Shallow modules**: complex interface, trivial implementation
- **Pass-through methods**: layers that add nothing, just forward calls
- Files longer than **500 lines** (flag), **1000 lines** (critical)
- Files shorter than **~50 lines** that are *not* obviously cohesive — may have been over-split or be junk fragments
- God modules doing too many things
- Related code not vertically close
- Inconsistent ordering (public API scattered among private helpers)
- **Information leakage**: implementation details exposed across module boundaries
- Circular or tangled dependencies between modules
- **If in doubt, split the crate** (M-SMALLER-CRATES): when a submodule could be used independently, it deserves to become its own crate. Splitting also improves compile times and prevents cyclic dependencies.

### Catch-All Module (Junk Drawer) — file-per-responsibility

> **Biomimetic frame — cellular differentiation.** A healthy eukaryotic cell has *specialized organelles*: a nucleus, mitochondria, ribosomes, lysosomes. Even the cell's garbage processor (lysosome) is a *specialized* organelle — there is no "misc bag organelle". An undifferentiated cell that keeps growing without specializing is a tumor. A `utils.rs` is exactly that: an undifferentiated file that grows because nobody decided where its contents belong. Every file should be a specialized organelle with one job.

**The rule:** one file = one cohesive responsibility. The file's name is a contract; if you can't name it without using `utils`, `helpers`, `common`, `misc`, `shared`, `lib`, `core`, or `tools`, you haven't decided what it does — and that indecision is technical debt.

**Detection (flag any of these):**
- File names matching `^(utils?|helpers?|common|misc|shared|tools|lib)\.(rs|ts|tsx|js|py|go|java)$`
- A file whose top-of-file doc comment contains "various", "miscellaneous", "and other", "etc."
- A file whose contents span > 2 unrelated domains (e.g., string formatting + HTTP retries + date parsing in the same file)
- A file < 50 lines that contains 3+ unrelated tiny helpers — should be **demixed** into the files those helpers serve

**Size guidance (cell-size analog):**

| Lines | Signal | Action |
|-------|--------|--------|
| < 50 | Likely under-grown or fragment | Consider merging into the file it primarily serves |
| 50–300 | Comfortable | Healthy cell |
| 300–500 | Watch | Look for emerging sub-responsibilities |
| 500–600 | Mitosis pressure | Plan a split |
| > 600 | Mitosis required | Split before adding more |

**Valid exception:** a small whole project (total source < 500 lines) may legitimately keep everything in one file. Over-splitting hurts readability as much as under-splitting.

**The 10-second findability test:** can a new developer find what they're looking for in under 10 seconds by reading file names alone? If not, the module structure has failed regardless of file size.

**Demixing principle (oil and water):** when contents lack semantic affinity, they will resist sharing the same file. Listen to that resistance — it's the codebase telling you to phase-separate.

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
- `#[allow(dead_code)]` or `#[allow(clippy::*)]` without justification — and **prefer `#[expect(...)]`** over `#[allow(...)]` (M-LINT-OVERRIDE-EXPECT). `#[expect]` warns if the marked lint *stops* triggering, preventing stale overrides accumulating in the codebase. Both forms should carry a `reason = "..."` attribute. `#[allow]` remains acceptable in generated code and macros.
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
- **Credential file permissions**: secret files (`.env`, `secret-*.yml`, `*credentials*`) should be mode 0600, not world-readable
- **`.env` not in `.gitignore`**: tracked `.env` files = secrets in version history
- **Raw secrets in config**: config files should reference secrets via env vars (`${VAR}`), not contain them inline
- **Secrets in git history**: `git log -p -- '*.env' '*.secret*'` may reveal committed secrets even if removed from HEAD

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
