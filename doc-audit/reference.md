# Documentation Audit Reference — Standards & Language Conventions

## Sources

### Standards
- **RFC 505** — Rust API Comment Conventions (2014): Summary line format, section ordering
- **RFC 1574** — More API Documentation Conventions (2016): Errors/Panics/Safety/Examples sections
- **Rust API Guidelines** — C-CRATE-DOC, C-EXAMPLE, C-FAILURE, C-LINK, C-HIDDEN checklist
- **Microsoft Pragmatic Rust Guidelines** (2025): M-DOC rules, 15-word summary, AI-optimized docs
- **Linux Kernel Rust Coding Guidelines**: SAFETY comments, tag format, Markdown in comments
- **Robert C. Martin** — *Clean Code* (2008): Comments explain WHY, not WHAT
- **Google Style Guide** — Documentation Best Practices: internal vs external doc division
- **Diátaxis** (Daniele Procida): 4-quadrant documentation structure

---

## Universal Rules (All Languages)

### Summary Line
- First line of doc comment = single sentence summary
- Max 15 words (Microsoft M-DOC-FIRST-SENTENCE)
- Functions: start with third-person singular verb ("Returns", "Creates", "Validates")
- Classes/structs/types: start with noun phrase ("A thread-safe container for...")
- Never start with "This function/method/class..."
- Never restate the item name as the summary

### Inline Comment Quality
- **WHY > WHAT**: Only comment things the code cannot express
- **No commented-out code**: Delete it; VCS has history
- **No closing brace comments**: `} // end if` is noise
- **No journal comments**: Changelogs belong in git, not file headers
- **Tag format**: `// TAG: Sentence with context, ending with period.`

### Tags Convention
| Tag | When | Expectation |
|-----|------|-------------|
| `TODO` | Planned improvement | Should reference issue/ticket when one exists |
| `FIXME` | Known bug or fragile code | Should reference issue/ticket |
| `HACK` | Intentional workaround | Must explain what it works around and when removable |
| `NOTE` | Non-obvious behavior | Explains a gotcha or constraint |
| `SAFETY` | Unsafe/unchecked operation | Explains why the operation is correct here |

### What Belongs Where
| Content | Location |
|---------|----------|
| API contract, usage, parameters, return values | Doc comments (public) |
| Why this implementation approach was chosen | Inline comment |
| Safety justification for unsafe/unchecked code | `// SAFETY:` before the block |
| Safety contract for callers | `# Safety` in doc comment |
| Architecture rationale | ADR (`docs/decisions/`) |
| How components interact | `docs/explanation/` |
| Step-by-step learning | `docs/tutorials/` |
| Solve a specific problem | `docs/how-to/` |
| Exhaustive option listing | `docs/reference/` |

---

## Language-Specific Conventions

### Rust

**Doc comment syntax**: `///` for items, `//!` for modules/crate root

**Summary line verbs**: "Returns", "Creates", "Sends", "Parses", "Converts"

**Required sections** (in this order):
1. Summary line
2. Extended description
3. `# Errors` — when function returns `Result`
4. `# Panics` — when function can panic
5. `# Safety` — when function is `unsafe`
6. `# Examples` — always plural, even with one example

**Clippy lints to reference**:
| Lint | What it catches |
|------|----------------|
| `missing_docs` | Public items without doc comments |
| `missing_errors_doc` | `Result`-returning fns without `# Errors` |
| `missing_panics_doc` | Panicking fns without `# Panics` |
| `missing_safety_doc` | `unsafe fn` without `# Safety` |
| `doc_markdown` | Technical identifiers not in backticks |

**Unsafe documentation pattern**:
```rust
/// Reads a response from the raw buffer.
///
/// # Safety
///
/// - `ptr` must point to a valid, aligned `Response`.
/// - The buffer must not be concurrently mutated.
pub unsafe fn read_response(ptr: *const u8) -> &Response { ... }

fn process() {
    // SAFETY: We validated alignment and length in `validate()` above.
    // The buffer is borrowed immutably, preventing concurrent mutation.
    unsafe { &*(buf.as_ptr() as *const Response) }
}
```

**Examples convention**:
- Use `?` operator, never `unwrap()` or `try!`
- Use `# ` prefix to hide boilerplate lines in rustdoc
- `no_run` attribute for examples requiring runtime resources

### TypeScript / JavaScript

**Doc comment syntax**: `/** ... */` (JSDoc/TSDoc)

**Summary line**: Same rules — first line is the summary

**Required tags**:
| Tag | When |
|-----|------|
| `@param` | Every parameter with non-obvious semantics |
| `@returns` | When return value needs explanation |
| `@throws` | When function can throw |
| `@example` | Public API functions |
| `@deprecated` | With migration path |
| `@see` | Cross-references |

**Type annotations in JSDoc** (JS only): `@param {string} name` — not needed in TypeScript

### Python

**Doc comment syntax**: Triple-quoted docstrings (`"""..."""`)

**Styles** (pick one per project, be consistent):
- **Google style**: Sections with indented content (`Args:`, `Returns:`, `Raises:`)
- **NumPy style**: Sections with underline (`Parameters`, `Returns`, `Raises`)
- **Sphinx/reST**: `:param name:`, `:returns:`, `:raises:`

**Required sections**:
| Section | When |
|---------|------|
| `Args` / `Parameters` | Non-trivial parameters |
| `Returns` | Non-obvious return value |
| `Raises` | When function raises exceptions |
| `Examples` | Public API, using doctest format |
| `Note` | Non-obvious behavior or gotchas |

### Go

**Doc comment syntax**: `//` preceding the declaration (no special marker)

**Conventions**:
- First sentence starts with the item name: `// Router dispatches requests to providers.`
- Package comment in `doc.go` or top of main file
- Use `//go:deprecated` for deprecated items
- Link to other symbols with `[Symbol]` syntax (Go 1.19+)

### Java / Kotlin

**Doc comment syntax**: `/** ... */` (Javadoc/KDoc)

**Required tags**:
| Tag | When |
|-----|------|
| `@param` | Every parameter |
| `@return` | Non-void methods |
| `@throws` / `@exception` | Checked and notable unchecked exceptions |
| `@see` | Cross-references |
| `@since` | Public API additions |
| `@deprecated` | With `@Deprecated` annotation and migration path |

### C# / .NET

**Doc comment syntax**: `///` with XML tags

**Required tags**:
| Tag | When |
|-----|------|
| `<summary>` | Every public member |
| `<param name="x">` | Every parameter |
| `<returns>` | Non-void methods |
| `<exception cref="T">` | When method can throw |
| `<example>` | Public API members |
| `<see cref="T"/>` | Cross-references to types/members |
| `<remarks>` | Extended description beyond summary |

---

## Documentation Completeness Index (DCI)

Quick scoring formula for audit reports:

```
DCI = (Σ wᵢ × sᵢ) / Σ wᵢ × 10

Where: wᵢ = weight (1-5), sᵢ = score (0.0 to 1.0)
```

**Scoring**: 0.0 = absent, 0.25 = stub, 0.5 = partial, 0.75 = good, 1.0 = excellent

**Grades**: A (9-10), B (7-8.9), C (5-6.9), D (3-4.9), F (0-2.9)

**Doc Debt** = (Undocumented Public Items / Total Public Items) × 100. Thresholds: 0-10% green, 10-25% yellow, 25-50% orange, 50%+ red.

---

## Coverage Tiers

Not all items need equal documentation depth:

| Tier | Items | Expected |
|------|-------|----------|
| **Critical** | Public API entry points, error types, config options | 100% with examples |
| **Important** | Public helpers, builders, conversion traits | 100% summary + params |
| **Nice-to-have** | Internal public items, test utilities | Summary line sufficient |
| **Skip** | Private items, test code, generated code | No doc comments needed |

---

## Confidence Protocol

When auditing Category 5 (Accuracy & Freshness), if you cannot verify with certainty (e.g., context limits, complex control flow), mark as `NEEDS-REVIEW` rather than `violation`. False positives erode trust in the audit.

---

## Anti-Patterns Checklist

These are the highest-value findings. Flag them as Critical:

| Anti-pattern | Why it's bad | Category |
|-------------|-------------|----------|
| Tautological doc ("Gets the name" on `get_name()`) | Zero information added | Summary Line |
| Stale doc (describes old behavior) | Actively misleading, worse than no doc | Accuracy |
| Copy-pasted doc on different functions | Will diverge and mislead | Accuracy |
| `// increment counter` above `count += 1` | Pure noise, trains readers to ignore comments | Inline |
| Commented-out code blocks | Clutter; VCS has history | Inline |
| Missing error docs on fallible public function | Callers can't handle errors properly | Sections |
| `unsafe` without SAFETY comment | Can't verify correctness during review | Sections |
| TODO without context/owner/issue | Undiscoverable tech debt | Inline |
| Inconsistent terminology across docs | Confuses readers about concepts | Accuracy |
| Examples that crash on error | Users copy examples verbatim | Sections |
| Tutorial mixed with reference | Reader doesn't know what mode they're in | Structure |
| How-to that explains instead of doing | Wastes reader's time when they're working | Structure |
