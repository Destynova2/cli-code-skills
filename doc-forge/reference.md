# Doc Forge Reference — Standards, Metrics & Language Conventions

## Sources & Influences

| Source | What we take from it |
|--------|---------------------|
| **OpenBSD mdoc(7)** | Docs are authoritative. Code changes require doc changes. Concise, structured, no fluff. |
| **Diátaxis** (Daniele Procida) | 4 quadrants: tutorial, how-to, reference, explanation. Never mix them. |
| **RFC 505 / 1574** | Rust doc comment conventions: summary line, canonical sections, examples. |
| **Microsoft Pragmatic Rust Guidelines** | M-DOC rules, 15-word summary, AI-optimized docs, `all.txt` for agents. |
| **Linux Kernel Coding Guidelines** | SAFETY comments, tag format, comments explain WHY not WHAT. |
| **Clean Code** (Robert C. Martin) | Comments are a failure to express in code. When needed, explain intent. |
| **llms.txt standard** (Jeremy Howard) | Markdown index of docs for LLM consumption. Two variants: summary + full. |
| **AGENTS.md / CLAUDE.md** | Project context files for AI coding agents. Progressive disclosure. Under 300 lines. |
| **roadmap.sh Technical Writer** | Skills taxonomy: research, content structure, user persona, progressive complexity. |

---

## Documentation Completeness Index (DCI) — Full Specification

### Formula

```
DCI = (Σ wᵢ × sᵢ) / Σ wᵢ × 10
```

Where each item `i` has:
- `wᵢ` = importance weight (1-5)
- `sᵢ` = completeness score (0.0 to 1.0)

### Scoring Rubric

| Score | Meaning | Criteria |
|-------|---------|----------|
| 0.0 | Absent | Item does not exist at all |
| 0.25 | Stub | Placeholder, empty template, or single sentence |
| 0.5 | Partial | Exists but missing significant content |
| 0.75 | Good | Covers most cases, minor gaps |
| 1.0 | Excellent | Complete, accurate, with examples |

### DCI Interpretation

| DCI | Grade | Meaning |
|-----|-------|---------|
| 9.0–10.0 | A | Exemplary — docs are a competitive advantage |
| 7.0–8.9 | B | Solid — usable, some gaps |
| 5.0–6.9 | C | Functional — gets the job done, needs work |
| 3.0–4.9 | D | Deficient — major gaps, users will struggle |
| 0.0–2.9 | F | Emergency — effectively undocumented |

### Documentation Debt Metric

```
Doc Debt % = (Undocumented Public Items / Total Public Items) × 100
```

Count as "public item":
- Rust: `pub fn`, `pub struct`, `pub enum`, `pub trait`, `pub type`, `pub mod`
- TypeScript/JS: exported functions, classes, types, interfaces
- Python: public functions (no leading `_`), classes, module-level variables
- Go: exported identifiers (capitalized)
- Java/Kotlin: `public` and `protected` methods, classes, interfaces

Count as "documented":
- Has a doc comment with at least a non-tautological summary line
- For fallible functions: also has error/exception documentation

### Coverage by Importance Tier

Not all items need equal documentation depth:

| Tier | Items | Expected coverage |
|------|-------|-------------------|
| **Critical** | Public API entry points, error types, config options | 100% with examples |
| **Important** | Public helpers, builders, conversion traits | 100% summary + params |
| **Nice-to-have** | Internal public items, test utilities | Summary line sufficient |
| **Skip** | Private items, test code, generated code | No doc comments needed |

---

## Audit Framework (6 Categories)

### Category 1: DOC COVERAGE (weight: 5)

Check for:
- Public API items missing doc comments
- Entry point / crate root / package root missing top-level docs
- Public modules missing module-level documentation
- Re-exports missing doc comments
- Exported constants/config values without documentation

### Category 2: SUMMARY LINE (weight: 3)

Rules (from RFC 505 + Microsoft M-DOC):
- First line = single sentence summary
- Max 15 words
- Functions: start with third-person singular verb ("Returns", "Creates", "Validates")
- Types/structs/classes: start with noun phrase ("A thread-safe container for...")
- Never start with "This function/method/class..."
- Never restate the item name

Anti-patterns:
- `get_name()` documented as "Gets the name" → tautological, zero information
- "This function returns the user" → drop "This function"
- Summary > 15 words → split into summary + extended description

### Category 3: STANDARD SECTIONS (weight: 4)

Required sections based on behavior:

| Section | Required when |
|---------|--------------|
| Errors / Exceptions | Function can fail (returns Result, throws, raises) |
| Panics / Throws | Function can crash/throw |
| Safety / Preconditions | Unsafe or unchecked operations |
| Parameters | Public functions with non-obvious params |
| Return values | Non-obvious semantics (what does null/None mean?) |
| Examples | Public API items (always) |

Examples must:
- Use proper error handling (`?` in Rust, try/catch in JS/TS)
- Never use `unwrap()`, `try!`, or unchecked casts
- Be runnable (or marked `no_run` with explanation)

### Category 4: INLINE COMMENTS (weight: 2)

Based on Clean Code + Linux Kernel style:

**Good comments** (keep these):
- WHY explanations: "We retry 3 times because the upstream API has transient failures"
- Constraint docs: "This must be called before init() due to ordering requirements in libfoo"
- Safety justifications: `// SAFETY: We validated alignment in validate() above`
- Consequence warnings: "Changing this timeout affects all downstream services"

**Bad comments** (flag these):
- WHAT comments: `// increment counter` above `counter += 1`
- Commented-out code: delete it, VCS has history
- Closing brace comments: `} // end if` is noise
- Journal comments: changelogs belong in git, not file headers
- Obvious comments: `// Constructor` above `fn new()`

**Tag format**: `// TAG: Sentence with context, ending with period.`

| Tag | When | Must include |
|-----|------|--------------|
| `TODO` | Planned improvement | Issue reference when one exists |
| `FIXME` | Known bug / fragile code | Issue reference |
| `HACK` | Intentional workaround | What it works around + when removable |
| `NOTE` | Non-obvious behavior | The gotcha or constraint |
| `SAFETY` | Unsafe/unchecked operation | Why it's correct here |

### Category 5: ACCURACY & FRESHNESS (weight: 5)

**Highest-value check. Actually read the implementation.**

Look for:
- Doc describes behavior that no longer matches implementation
- Parameters documented that no longer exist
- New parameters not documented
- Return value docs that are wrong
- Copy-pasted documentation between different functions
- Inconsistent terminology (same concept, different names)
- Outdated examples using removed functions or old API patterns

**Confidence protocol**: If you cannot verify with certainty (e.g., context window limits), flag as `NEEDS-REVIEW` rather than `violation`. False positives erode trust.

### Category 6: CROSS-REFERENCES & LINKING (weight: 2)

Check for:
- Type/class names in prose without intra-doc links
- Related items not cross-referenced (builder without linking to what it builds)
- Broken or dangling references
- Bare URLs not wrapped in proper link format

---

## Language-Specific Conventions

### Rust

**Doc syntax**: `///` for items, `//!` for modules/crate root

**Section order**: Summary → Extended → `# Examples` → `# Errors` → `# Panics` → `# Safety` → `# Abort`

**Clippy lints to enable**:
- `missing_docs`: public items without doc comments
- `missing_errors_doc`: Result-returning fns without `# Errors`
- `missing_panics_doc`: panicking fns without `# Panics`
- `missing_safety_doc`: unsafe fn without `# Safety`
- `doc_markdown`: identifiers not in backticks

**Unsafe pattern**:
```rust
/// Reads a response from the raw buffer.
///
/// # Safety
///
/// - `ptr` must point to a valid, aligned `Response`.
/// - The buffer must not be concurrently mutated.
pub unsafe fn read_response(ptr: *const u8) -> &Response { ... }

fn process() {
    // SAFETY: We validated alignment and length in validate() above.
    // The buffer is borrowed immutably, preventing concurrent mutation.
    unsafe { &*(buf.as_ptr() as *const Response) }
}
```

**Examples**: Use `?` operator, `# ` prefix to hide boilerplate, `no_run` for runtime resources.

### TypeScript / JavaScript

**Doc syntax**: `/** ... */` (JSDoc/TSDoc)

**Required tags**: `@param` (non-obvious params), `@returns`, `@throws`, `@example`, `@deprecated` (with migration path), `@see`

**No `@param` type annotations in TypeScript** — the type system handles that.

### Python

**Doc syntax**: `"""..."""` (Google, NumPy, or Sphinx style — pick one, be consistent)

**Sections**: `Args:`, `Returns:`, `Raises:`, `Examples:` (doctest format), `Note:`

### Go

**Doc syntax**: `//` preceding declaration. First sentence starts with item name.

**Package comment** in `doc.go`. Link symbols with `[Symbol]` (Go 1.19+).

### Java / Kotlin

**Doc syntax**: `/** ... */` (Javadoc/KDoc)

**Required**: `@param` (every param), `@return` (non-void), `@throws`, `@see`, `@since`, `@deprecated`

### C# / .NET

**Doc syntax**: `///` with XML tags

**Required**: `<summary>`, `<param name="x">`, `<returns>`, `<exception cref="T">`, `<example>`

---

## LLM Documentation Standards

### AGENTS.md Best Practices

Synthesized from Anthropic, HumanLayer, and community research:

1. **Under 300 lines.** Every line competes for attention with actual work.
2. **One sentence project description.** Acts as a role-based prompt.
3. **Describe capabilities, not file paths.** Paths change; capabilities don't.
4. **Domain concepts > file structure.** "Organization" vs "Workspace" is stable; `src/auth/handlers.ts` is not.
5. **Exact commands.** `pnpm test:integration`, not "run the tests".
6. **No style guides.** Let the linter do the linter's job. LLMs are in-context learners.
7. **Progressive disclosure.** Point to `docs/` for deep dives. Don't inline everything.
8. **Gotchas only.** Document what the agent would get wrong, not what it'd get right.

### llms.txt Standard

From llmstxt.org:

```markdown
# Project Name

> One paragraph summary

## Docs
- [Title](url): One-line description

## Optional
- [Source overview](url): Key source files
- [API spec](url): OpenAPI/AsyncAPI if applicable
```

Two variants:
- `llms.txt`: compact index with links (for large docs)
- `llms-full.txt`: all content inline (for small projects, <50K tokens)

### Microsoft all.txt Approach

All guidelines concatenated in a single flat file with `---` separators between sections. Each section has:
- Heading with rule ID
- `<why>` tag explaining rationale
- `<version>` tag for tracking changes
- Code examples inline

This format is optimized for LLM consumption: flat, searchable, self-contained.

---

## Anti-Patterns Checklist (Critical Findings)

These are the highest-impact findings. Always flag as Critical:

| Anti-pattern | Impact | Category |
|--------------|--------|----------|
| Tautological doc ("Gets the name" on `get_name()`) | Zero information added | Summary |
| Stale doc (describes old behavior) | Actively misleading — worse than no doc | Accuracy |
| Copy-pasted doc on different functions | Will diverge and mislead | Accuracy |
| `// increment counter` above `count += 1` | Trains readers to ignore ALL comments | Inline |
| Commented-out code blocks | Clutter, VCS has history | Inline |
| Missing error docs on fallible public function | Callers can't handle errors properly | Sections |
| `unsafe` without SAFETY comment | Can't verify correctness in review | Sections |
| TODO without context/owner/issue | Invisible tech debt | Inline |
| Inconsistent terminology across docs | Confuses readers about concepts | Accuracy |
| Examples that crash on error | Users copy examples verbatim | Sections |
| Tutorial mixed with reference | Reader doesn't know what mode they're in | Structure |
| How-to that explains instead of doing | Wastes reader's time when they're working | Structure |

---

## Audit Report Format

```markdown
# Documentation Audit Report

**Project**: [name]
**Target**: [file/directory audited]
**Language(s)**: [detected]
**Date**: [date]
**DCI Score**: X.X/10 (Grade: X)
**Doc Debt**: X%

## Scores by Category

| # | Category | Score | Critical | Flags |
|---|----------|-------|----------|-------|
| 1 | Doc Coverage | X/10 | N | N |
| 2 | Summary Lines | X/10 | N | N |
| 3 | Standard Sections | X/10 | N | N |
| 4 | Inline Comments | X/10 | N | N |
| 5 | Accuracy & Freshness | X/10 | N | N |
| 6 | Cross-references | X/10 | N | N |

## Critical Violations (must fix)

### [Category]: [violation title]
- **File**: `path/to/file:line`
- **What**: [description]
- **Rule**: [which standard it violates]
- **Fix**: [concrete suggestion]

## Flags (should fix)

[same format, grouped by category]

## Good Practices Found

[highlight documentation done well — positive reinforcement matters]

## Recommended Next Steps

1. [highest-impact, lowest-effort fix]
2. [second]
3. [third]

## DCI Breakdown

[Full checklist table with scores]
```

---

## Content Strategy Matrix

Use this to decide what to generate based on project characteristics:

| Project Type | Must Have | Should Have | Nice to Have |
|-------------|-----------|-------------|--------------|
| **CLI tool** | README, `--help` text, AGENTS.md | How-to guides, config reference | Tutorials, architecture |
| **Library/SDK** | API reference, examples, AGENTS.md, llms.txt | Tutorials, error guide | Architecture, ADRs |
| **Web service/API** | API docs, deployment guide, AGENTS.md | Tutorials, config ref, security | ADRs, troubleshooting |
| **Platform/Framework** | Full Diátaxis suite, AGENTS.md, llms.txt | Everything | - |
| **Internal tool** | README, AGENTS.md, how-to | Config ref, troubleshooting | Architecture |
| **Script (<500 LOC)** | Header comment + README | AGENTS.md | - |
