# Clean Code Reference — Full Principles Catalog

## Sources

This reference synthesizes principles from multiple authorities, not just Robert C. Martin.

### Books
- **Robert C. Martin** — *Clean Code* (2008): Naming, functions, comments, formatting, error handling
- **John Ousterhout** — *A Philosophy of Software Design* (2018): Deep vs shallow modules, cognitive load, complexity management
- **Martin Fowler** — *Refactoring* (2018 2nd ed): Code smells catalog, refactoring patterns
- **Kent Beck** — *Implementation Patterns* (2007): Communication through code
- **Michael Feathers** — *Working Effectively with Legacy Code* (2004): Seam model, characterization tests

### Papers & Metrics
- **SonarSource** — Cognitive Complexity metric (2017): Beyond cyclomatic complexity
- **Casey Muratori** — *Clean Code, Horrible Performance* (2023): Performance cost of over-abstraction

---

## Principles Not in the wojteklu Gist

### From Ousterhout: Complexity Management

1. **Deep modules > Shallow modules**
   - A deep module has a simple interface but powerful functionality
   - A shallow module has a complex interface relative to what it does
   - Example: Unix file I/O (`open/read/write/close`) is deep — 4 functions, enormous power
   - Anti-example: A `StringHelper` class with 50 trivial wrappers is shallow

2. **Define errors out of existence**
   - Instead of throwing exceptions for edge cases, design APIs that can't fail
   - Example: `delete(file)` should succeed even if file doesn't exist (idempotent)
   - In Rust: prefer `Option<T>` returns over panicking when absence is expected

3. **Different layer, different abstraction**
   - If adjacent layers have similar abstractions, there's probably a missing or unnecessary layer
   - Pass-through methods/functions are a red flag

4. **Strategic vs tactical programming**
   - Tactical: "just make it work" — accumulates complexity
   - Strategic: invest 10-20% extra time in good design — pays off within weeks

5. **Pull complexity downward**
   - Better for a module to be internally complex with a simple interface
   - Don't push complexity onto callers to "keep the implementation simple"

### From Fowler: Code Smells Catalog

6. **Feature envy**: A function uses another module's data more than its own
7. **Data clumps**: Same 3+ fields always appear together (should be a struct)
8. **Primitive obsession**: Using `String`/`u32` where a domain type would be clearer
9. **Divergent change**: One module changes for multiple unrelated reasons
10. **Shotgun surgery**: One change requires edits across many modules
11. **Message chains**: `a.get_b().get_c().get_d()` — Law of Demeter violation
12. **Middle man**: Class that delegates everything without adding value
13. **Speculative generality**: Hooks/parameters/abstractions for "future needs" that never come
14. **Temporary field**: Instance variable only set in some circumstances

### Modern Principles (post-Martin)

15. **Immutability by default**
    - Mutable state is the #1 source of bugs in concurrent systems
    - In Rust: prefer `let` over `let mut`, prefer `&T` over `&mut T`
    - Only make mutable what must change

16. **Pure functions where possible**
    - Same inputs always produce same outputs, no side effects
    - Easier to test, reason about, parallelize
    - Separate pure computation from effectful I/O at module boundaries

17. **Parse, don't validate** (Alexis King, 2019)
    - Instead of validating data then passing raw strings/numbers around, parse into typed representations immediately
    - Example: parse `"user@email.com"` into `Email(String)` at the boundary, not `validate_email(s: &str) -> bool`

18. **Make illegal states unrepresentable** (Yaron Minsky)
    - Use the type system to prevent invalid combinations
    - In Rust: enums > boolean flags, newtype wrappers > raw primitives
    - Example: `enum ConnectionState { Connected(Stream), Disconnected }` not `struct Connection { stream: Option<Stream>, is_connected: bool }`

19. **Fail fast, fail loud**
    - Detect errors as close to the source as possible
    - Don't let invalid state propagate through the system
    - In Rust: validate at construction time (constructor validates, fields stay private)

20. **Boring technology** (Dan McKinley)
    - Use proven, well-understood tools/patterns
    - Save innovation budget for your actual problem domain
    - Every exotic dependency is a maintenance liability

### Cognitive Complexity (SonarSource)

21. **Cognitive complexity scoring**
    - +1 for each `if`, `else if`, `else`, `for`, `while`, `match` arm, `&&`, `||`
    - +1 **extra** for each level of nesting (the key difference from cyclomatic)
    - Break/continue to labels: +1
    - Recursion: +1
    - Sequences of `&&`/`||` don't increment (treated as single decision)
    - **Target: < 15 per function**

### Performance-Aware Clean Code (Muratori)

22. **Don't abstract for abstraction's sake**
    - Virtual dispatch (trait objects) has real cost in hot paths
    - Excessive layering can defeat CPU cache and branch prediction
    - Clean code should be *simple*, not necessarily *maximally decomposed*
    - A flat 20-line function can be cleaner than 5 nested 4-line functions

23. **Data-oriented thinking**
    - Sometimes the cleanest solution is to restructure data, not add more functions
    - Struct-of-arrays vs array-of-structs matters for performance
    - Profile before abstracting hot paths

---

## Rust-Specific Clean Code Idioms

### Ownership & Borrowing
- Prefer `&str` over `&String`, `&[T]` over `&Vec<T>` in function params
- Use `Cow<str>` when a function might or might not need to allocate
- Avoid unnecessary `.clone()` — if you clone to satisfy the borrow checker, reconsider the design

### Error Handling
- Use `thiserror` for library errors, `anyhow` for application errors
- `?` operator over manual `match` on `Result`
- Custom error types over `String` errors
- `unwrap()`/`expect()` only when invariant is proven (with comment) or in tests

### Pattern Matching
- Prefer `if let` over `match` with single arm + `_ =>`
- Use `matches!()` macro for boolean match checks
- Exhaustive matching (no `_ =>` wildcard) when enum variants should be handled explicitly

### Iterators
- `.iter().map().collect()` over manual `for` + `push`
- Chain iterator adaptors over intermediate collections
- `for x in &collection` over `for i in 0..collection.len()` (unless index is needed)

### Type System
- Newtype pattern for domain concepts (`struct UserId(u64)`)
- `#[must_use]` on functions whose return value shouldn't be ignored
- `#[non_exhaustive]` on public enums to allow future variants

### Dead Code
- Never use `#[allow(dead_code)]` as a permanent solution
- If code is for future use, delete it. Write it again when needed.
- `#[cfg(test)]` for test-only helpers

### Clippy
- `#[allow(clippy::*)]` requires a `// Reason:` comment explaining why
- Run `clippy -- -D warnings` in CI
- Don't blanket-allow at crate level
