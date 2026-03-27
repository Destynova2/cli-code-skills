# Three-Layer Verification — Detailed Checks

> **When to read:** During Step 3 when running the verification layers.

---

## Layer 1 — Structural (lychee-style)

Check that every reference in docs points to something real.

| Check | How | Severity |
|-------|-----|----------|
| Internal file links (`[text](path/to/file)`) | Glob for the target file, verify it exists | Critical |
| Anchor links (`[text](#section)`) | Parse target file for matching heading | Warning |
| Image references (`![alt](path/to/image)`) | Verify image file exists | Critical |
| External URLs | Skip by default (network-dependent). Flag `[TODO]` or placeholder URLs | Info |
| Code block file references (`src/main.rs`, `config.yaml`) | Grep for mentioned filenames in the project | Warning |
| Import/use statements mentioned in docs | Verify the module/package/crate exists | Critical |

**How to run:**

1. Glob all documentation files: `**/*.md`, `docs/**/*`
2. Extract all links using regex: `\[.*?\]\((.*?)\)`, `!\[.*?\]\((.*?)\)`
3. For each link, verify the target exists relative to the doc file's location
4. Collect results as: `{file, line, link, status: found|broken|ambiguous}`

---

## Layer 2 — Semantic (DocPrism/Swimm-style)

Check that names, concepts, and claims in docs match the actual code.

| Check | How | Severity |
|-------|-----|----------|
| Module/package names | Extract names from docs, grep in source tree | Critical |
| Function/method names | Extract `function_name()` patterns from docs, grep in code | Critical |
| Type/struct/class names | Extract PascalCase names from docs, grep in code | Critical |
| Endpoint paths (`/api/v1/users`) | Extract URL patterns from docs, grep in route definitions | Critical |
| CLI commands/flags | Extract backtick commands from docs, grep in CLI definition code | High |
| Environment variables | Extract `ENV_VAR` patterns from docs, grep in code | High |
| Config keys | Extract config references from docs, verify in config files | Warning |
| Dependency names | Extract package names from docs, verify in manifest | Warning |
| Version numbers | Extract versions from docs, compare with manifest | Warning |
| File counts / feature counts | Extract numeric claims ("5 endpoints"), count actual | Warning |

**Methodology (inspired by DocPrism's LCEF):**

1. **Local Categorization** — For each doc file, extract all code-referencing tokens:
   - Backtick-wrapped identifiers: `` `function_name` ``
   - PascalCase words in prose (likely type names)
   - UPPER_SNAKE_CASE words (likely constants/env vars)
   - Path-like strings (`src/`, `/api/`, `config/`)
   - Command-like strings starting with `$`, `./`, or common CLI tools

2. **External Filtering** — For each extracted token:
   - Search in source code (grep)
   - Search in manifest files
   - Search in config files
   - Classify: `found` | `not_found` | `renamed` (fuzzy match) | `generic` (skip common words)

3. **Drift Detection** — Check git blame/log for recently changed files:
   - If a source file was modified recently but its doc references weren't → flag as potential drift
   - If a doc file references a deleted file → flag as stale

**Terminology Consistency:**

Scan all docs for the same concept referred to differently:
- "user" vs "client" vs "account" vs "customer" for the same entity
- "service" vs "server" vs "backend" vs "API" for the same component
- Build a terminology map and flag inconsistencies

---

## Layer 3 — Executable (Docs-as-Tests-style)

Check that commands and examples in docs actually work.

| Check | How | Severity |
|-------|-----|----------|
| Shell commands in README (`` ```bash ``) | Parse code blocks, classify as runnable vs illustrative | Critical |
| Installation commands | Verify the install path exists or the package is published | Critical |
| Build commands (`cargo build`, `npm install`) | Dry-run if safe (check syntax, not execute) | High |
| Example code snippets | For Rust: check if they'd compile (type/import references). For others: syntax check | Warning |
| Expected output shown in docs | If docs show `→ output`, verify format is plausible | Info |

**Safety rules:**
- **NEVER execute** commands that modify state (install, write, delete)
- **Dry-run only**: verify syntax, check that referenced files/tools exist
- **Classify commands** before checking:
  - `safe_to_verify`: `ls`, `cat`, `grep`, `--version`, `--help`
  - `check_syntax_only`: `cargo build`, `npm install`, `docker run`
  - `skip`: `rm`, `sudo`, `curl -X POST`, anything with side effects

**Mermaid diagram verification:**

For each Mermaid block in docs:
1. Parse node names from the diagram
2. Grep for those names in source code
3. Flag nodes that don't correspond to real files/modules/services
4. Flag edges (relationships) that don't match import/call patterns
