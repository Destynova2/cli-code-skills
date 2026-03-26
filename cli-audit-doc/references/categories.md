# Doc Audit Categories — Detailed Checks

> **When to read:** Step 3 (Score all dimensions). Contains the full check lists per category.

---

## C1: DIATAXIS COVERAGE (10%)

**Source:** Daniele Procida (Diataxis framework)

Check if the project has all four doc types:
- **Tutorials** (learning-oriented): guided first steps, progressive
- **How-to Guides** (task-oriented): "How to [verb] [object]"
- **Reference** (information-oriented): exhaustive, factual, mirrors code structure
- **Explanation** (understanding-oriented): "why" questions, ADRs, architecture decisions

Also check **mode purity**: each document stays in its lane. A tutorial that suddenly becomes a reference table is a violation.

## C2: COMPLETENESS (12%)

**Source:** RFC 1574, rustdoc, Google dev docs

Check for:
- Public API items missing doc comments (functions, classes, structs, types, interfaces, exports)
- Entry point / root module missing top-level documentation
- Public modules missing module-level documentation
- Re-exports or public aliases missing doc comments
- Exported constants/config values without documentation
- **API coverage ratio**: % of public items with doc comments (target: > 80%)

## C3: FRESHNESS & ACCURACY (10%)

**Source:** Swimm, docs-as-tests methodology

Check for:
- Doc comment describes behavior that **no longer matches** implementation
- Parameters documented that **no longer exist**, or new parameters not documented
- Return value docs that are wrong
- Copy-pasted documentation between distinct functions
- Inconsistent terminology (same concept called different names)
- Outdated examples using old API patterns
- **Doc-code drift**: time delta between last code change and last doc change for the same module

## C4: READABILITY & PROSE QUALITY (10%)

**Source:** Google dev docs style guide, Microsoft Writing Style Guide, Vale linter

Check for:
- **Weasel words**: "various", "many", "several", "some" without specifics
- **Condescending language**: "simply", "just", "obviously", "easy"
- **Passive voice overuse**: > 20% passive sentences in a doc
- **Sentence length**: average > 25 words per sentence
- **Wall of text**: paragraphs > 5 sentences without structure (headings, lists, code)
- **Tautological summaries**: restating the name ("Gets the name" on `get_name()`)
- Function summaries not starting with third-person verb ("Returns", "Creates")
- Type summaries not starting with a noun phrase ("A thread-safe...", "An iterator over...")

## C5: STRUCTURE & FINDABILITY (8%)

**Source:** Diataxis, information architecture principles

Check for:
- Heading hierarchy (no skipping levels, H1 → H2 → H3)
- **Scanability**: headers, bullet points, short paragraphs
- **Front-loading**: most important info first (inverted pyramid)
- Action-oriented headings ("Install the CLI" not "Installation")
- **Orphan pages**: docs not linked from anywhere, unreachable
- Cross-references between related items
- Index or glossary for large doc sets

## C6: STANDARD SECTIONS (8%)

**Source:** RFC 1574 (Rust), language-specific conventions

Check for required sections based on the code's behavior:
- **Errors/Exceptions**: functions that fail without documenting error conditions
- **Panics/Throws**: functions that crash without documenting when
- **Safety/Preconditions**: unsafe operations without contract documentation
- **Parameters**: non-obvious parameters left undocumented
- **Return values**: semantics not documented (what does null/None/empty mean?)
- **Examples**: public API items without usage examples
- Examples using crash-prone patterns (`unwrap()`, unchecked casts) instead of proper error handling

## C7: CODE EXAMPLES (10%)

**Source:** Rust doc-tests, Google "working code samples" principle

Check for:
- Public API items without any example
- Examples that can't compile/run (syntax errors, missing imports)
- Examples using `unwrap()` instead of proper error handling
- Examples not showing the most common use case
- **Doc-tests** (Rust): examples that are actually tested as part of CI
- Copy-pasteable examples (include all necessary imports/setup)

## C8: ACCESSIBILITY & INCLUSIVITY (6%)

**Source:** Google dev docs, WCAG, Microsoft bias-free language

Check for:
- Alt text for images
- Color not used as the only differentiator
- Bias-free language (avoid "master/slave", "whitelist/blacklist", gendered pronouns)
- Consistent terminology across the project
- **Global readiness**: no untranslatable idioms, culturally specific humor

## C9: INLINE COMMENTS QUALITY (8%)

**Source:** Clean Code, Linux Kernel coding style

Check for:
- Redundant comments restating the code
- Commented-out code (should be deleted, VCS has history)
- Stale TODOs/FIXMEs without tracking (issue reference, date, owner)
- Comments explaining WHAT instead of WHY
- Missing constraint docs for non-obvious business logic, magic numbers
- Missing safety justifications for unsafe/unchecked blocks

## C10: CROSS-REFERENCES & LINKING (5%)

Check for:
- Type/class names mentioned in prose without linking
- Related items not cross-referenced (builder without linking to what it builds)
- Broken or dangling references
- Bare URLs not wrapped in markdown links

## C11: TESTING & CI (8%)

**Source:** docs-as-code movement, Write the Docs community

Check for:
- Are doc-tests enabled and running? (Rust: `cargo test --doc`, Python: `doctest`)
- Link checking in CI (lychee, markdown-link-check)
- Prose linting in CI (Vale, markdownlint)
- Spell checking in CI (cspell)
- **Living documentation**: is there any automated verification that docs match code?

## C12: MAINTENANCE PROCESS (5%)

Check for:
- Doc changes reviewed like code (PR-based workflow)
- Documentation included in Definition of Done
- Ownership clear (who maintains which docs?)
- Deprecation process (old docs marked, not silently removed)
- Update cadence matches release cadence
