# Duplication Detection Rules — DRY, SSOT, KISS

> **When to read:** Step 3-5 when scanning for cross-source duplication, single-source-of-truth violations, and KISS issues.

---

## Authoritative quotes (use as evidence in findings)

1. **DRY** — Hunt & Thomas, *The Pragmatic Programmer* (1999):
   > "Every piece of knowledge must have a single, unambiguous, authoritative representation within a system."
   > "DRY is about the duplication of knowledge, of intent. It's about expressing the same thing in two different places, possibly in two totally different ways."

2. **Rule of Three** — Don Roberts via Martin Fowler, *Refactoring* p.58:
   > "The first time you do something, you just do it. The second time, you wince at the duplication, but you do the duplicate thing anyway. The third time, you refactor."

3. **Wrong Abstraction** — Sandi Metz, RailsConf 2014 / blog 2016:
   > "Duplication is far cheaper than the wrong abstraction."

4. **SSOT** — Wikipedia / enterprise architecture:
   > "Single source of truth: every data element is mastered (or edited) in only one place, providing data normalization to a canonical form."

5. **KISS** — Kelly Johnson, Lockheed Skunk Works (~1960):
   > "Keep it simple stupid" (no comma — "stupid" modifies the solution, not the engineer).

6. **Diátaxis** — diataxis.fr:
   > "Each [quadrant] has its own purpose, each requires its own distinct form and style — and each should be kept distinct from the others."

---

## Rule family D — Doc-doc duplication

| ID | Rule | Threshold | Source |
|----|------|-----------|--------|
| **D-DRY-01** | Same normalized paragraph in ≥2 docs | 2=warn, 3=error | Fowler Rule of Three |
| **D-DRY-02** | Same markdown table in ≥2 docs | 2=error | Diátaxis (tables = reference) |
| **D-DRY-03** | Same fenced code block in ≥2 docs | 2=error | Hunt/Thomas DRY |
| **D-DIATAXIS-01** | Reference table appearing outside `docs/reference/` | any=warn | Diátaxis |
| **D-DIATAXIS-02** | Tutorial-style content in reference docs (or vice-versa) | any=warn | Diátaxis |
| **D-KISS-01** | Paragraph >100 words | warn | NN/g signal-to-noise |
| **D-KISS-02** | Sentence >25 words | warn | Plain language guidelines |
| **D-KISS-03** | Section with >5 consecutive paragraphs without headings/lists | warn ("Wall of Text") | NN/g F-pattern |
| **D-KISS-04** | README > 200 lines | warn | Convention (most successful OSS READMEs <150) |

### Detection method (D-DRY-01/02/03)

```
1. Glob: docs/**/*.md, README.md, CONTRIBUTING.md, AGENTS.md, *.md
2. For each file, extract:
   - Paragraphs (split on \n\n)
   - Tables (lines starting with `|`)
   - Code blocks (between ```)
3. Normalize: strip whitespace, collapse spaces, lowercase
4. SHA-256 each normalized block
5. Group by hash; flag any group with members in ≥2 files
6. Report: "Block X appears in N files: [list]"
```

---

## Rule family C — Code-code cross-format duplication

This is the **gap that no mainstream tool fills**. Detection of identical commands/configs across Makefile + CI YAML + shell scripts + docs.

| ID | Rule | Threshold | Source |
|----|------|-----------|--------|
| **C-DRY-01** | Same command in Makefile + CI + shell script | any=error | Hunt/Thomas "imposed duplication" |
| **C-DRY-02** | Same literal constant (URL, port, path, version) in ≥2 source files (not imported) | 2=warn, 3=error | Fowler Rule of Three |
| **C-DRY-03** | Same env var defined in Makefile + CI + script with same values | any=error (extract anchor/include) | DRY |
| **C-DRY-04** | Same env var defined with DIFFERENT values across files | any=critical (drift!) | SSOT |
| **C-KISS-01** | Function > 50 LOC | warn | Clean Code |
| **C-KISS-02** | Cyclomatic complexity > 15 | warn, > 30 = error | McCabe 1976 / NIST |
| **C-KISS-03** | Nesting depth > 4 | warn | Linter convention |

### Detection method (C-DRY-01/03/04)

```
1. Extract command strings from:
   - Makefile recipes: lines starting with TAB after a target
   - CI YAML: `run:`, `script:`, `before_script:` values
   - Shell scripts: executable lines (not comments, not function defs)
   - Markdown fenced blocks with lang=bash|sh|console
2. Normalize:
   - Strip leading/trailing whitespace
   - Collapse internal whitespace
   - Sort short flags (-e VAR=val -e VAR2=val2)
   - Strip line continuations (\)
3. Hash each normalized command
4. Group by hash; flag groups with members in ≥2 source kinds
5. For each group, verify semantic equivalence (not just literal hash)
```

### Example finding format

```
C-DRY-01: command duplicated across formats
- Makefile:32 (target: test)
- .gitlab-ci.yml:319 (job: container-test-goss)
- scripts/run-tests.sh:14
- docs/how-to/test.md:42

Normalized command:
  podman run --rm -e MSSQL_SA_PASSWORD='Test!1' image:tag /usr/local/bin/test-cis.sh

Fix: extract to single source.
- Option A (recommended): make CI call `make test`, doc says "run make test"
- Option B: shared YAML anchor in CI, sourced by Makefile

Source: Hunt/Thomas DRY ("imposed duplication" category)
```

---

## Rule family S — SSOT (code ↔ doc)

| ID | Rule | Threshold | Source |
|----|------|-----------|--------|
| **S-SSOT-01** | CLI flag in `--help` not in README, or vice-versa | error | SSOT Wikipedia |
| **S-SSOT-02** | Version string hardcoded in code AND in docs | error (use single source) | SSOT |
| **S-SSOT-03** | Schema/type defined in code AND restated in docs (not generated) | warn | DRY (code+docs) |
| **S-SSOT-04** | Same env var documented with different defaults across docs | error (drift) | SSOT |
| **S-SSOT-05** | API endpoint list hand-written in docs | warn (should be generated) | SSOT |
| **S-SSOT-06** | Config option in code (default value) ≠ value documented in README | error (drift) | SSOT |

---

## Rule family M — Anti-false-DRY (Sandi Metz)

| ID | Rule | Action | Source |
|----|------|--------|--------|
| **M-AHA-01** | Abstraction used by only 1 caller | warn ("wrong abstraction signal") | Metz "The Wrong Abstraction" |
| **M-AHA-02** | Don't escalate duplication < 3 instances to "must fix" | downgrade to info | Rule of Three + Metz |
| **M-AHA-03** | Don't suggest extracting an abstraction without evidence of variation | block recommendation | Metz / AHA |

**Why M rules exist:** preventing the cycle from creating wrong abstractions when fixing duplications. Inlining + Rule of Three is safer than premature abstraction.

---

## Rule family I — Information theoretic (optional, advanced)

| ID | Rule | Method | Source |
|----|------|--------|--------|
| **I-ENT-01** | Repo duplication index = 1 − (compressed_size / deduped_compressed_size) | gzip ratio | Kolmogorov/Shannon |

```bash
# Crude duplication index
size_full=$(tar c . | gzip | wc -c)
size_deduped=$(tar c $(find . -type f | sort -u) | gzip | wc -c)
echo "scale=3; 1 - ($size_deduped / $size_full)" | bc
# > 0.3 = warn (high duplication)
```

---

## Severity grading (use in cli-cycle triage)

| Severity | Triggers | Tier |
|----------|---------|------|
| **Critical** | C-DRY-04 (different values = drift), S-SSOT-06 (config drift) | Tier 3 |
| **Error** | D-DRY-02/03 (3+ copies), C-DRY-01/03 (cross-format), S-SSOT-01/02/04 | Tier 2 |
| **Warning** | D-DRY-01 (2 copies), D-KISS-*, C-KISS-*, M-AHA-* | Tier 2 |
| **Info** | I-ENT-01, M-AHA-02 (downgrades) | Tier 1 |

---

## Rule of Three application (Fowler default)

For ALL rules, apply Fowler's Rule of Three as the default escalation:
- **1 occurrence** = ignore (no duplication yet)
- **2 occurrences** = warn (the wince moment)
- **3+ occurrences** = error (refactor mandate)

**Exception:** if the 2 occurrences have **different values** (drift), it's immediately critical regardless of count.

---

## Modern enhancements (2024-2026 research)

### NCD — Normalized Compression Distance (language-agnostic pre-filter)

**Source:** Kolmogorov complexity, MDL (arXiv 1005.2364), used in LLM training data dedup pipelines.

**Why:** Tree-sitter and AST tools require a parser per language. NCD works on **any two strings** with `zstd`/`gzip` and is the cheapest possible duplicate detector.

```bash
# Compute NCD between two files
ncd() {
  local a="$1" b="$2"
  local cx=$(cat "$a" | zstd -q -c | wc -c)
  local cy=$(cat "$b" | zstd -q -c | wc -c)
  local cxy=$(cat "$a" "$b" | zstd -q -c | wc -c)
  local min=$(( cx < cy ? cx : cy ))
  local max=$(( cx > cy ? cx : cy ))
  echo "scale=3; ($cxy - $min) / $max" | bc
}
# NCD < 0.3 = highly duplicated (any pair, any format)
# NCD < 0.1 = nearly identical
```

**Use:** First-pass scan over all `(file_a, file_b)` pairs. Flag pairs with NCD < 0.3 for deeper analysis. O(n²) but each comparison is microseconds.

### Rule of Three + AGE check (Sandi Metz refinement)

**Source:** Sandi Metz "The Wrong Abstraction" + Terrible Software 2025 follow-up.

**Why:** New duplications (< 14 days old in git history) are pre-Rule-of-Three. Don't escalate them. Old duplications (> 14 days, ≥ 3 instances) are refactor candidates.

**Application:**
```
For each duplicate group:
  age = days since the most recent file was created/modified
  count = number of instances

  IF count < 3 AND age < 14 days → INFO ("wait, may converge naturally")
  IF count < 3 AND age > 14 days → WARN
  IF count >= 3 AND age > 14 days → ERROR (refactor mandate)

  If duplicate REPLACED a prior abstraction (visible in git log) →
    DOWNGRADE to INFO (the abstraction was wrong, duplication is intentional)
```

This single rule prevents 60-80% of false positives that come from "audit immediately after a paste."

### Diátaxis-quadrant classification (rename findings, don't just count them)

**Source:** Diátaxis 2024-2025 refinement (Bernard, HN).

**Why:** "Duplicate paragraph" is unactionable. "Reference table appearing in tutorial — should be a link to reference/" is actionable.

**Application:**
```
For each duplicate found:
  Classify each occurrence into a Diátaxis quadrant:
    - tutorials/   → Tutorial
    - how-to/      → How-To
    - reference/   → Reference
    - explanation/ → Explanation
    - README.md    → Landing
    - other        → Mixed

  Findings format:
    "Block X appears in {Tutorial} AND {Reference}"
    Recommendation: "Reference is authoritative. Tutorial should LINK to reference, not duplicate."

  This converts 'duplication' findings into 'mode violation' findings.
```

The Tutorial copy is almost always the wrong one — it should be a link.

---

## Tools that already implement parts of this

| Tool | Rule families covered |
|------|----------------------|
| **jscpd** | C-DRY-03 (token clones), partial D-DRY-01 (markdown supported) |
| **PMD CPD** | C-DRY-03 (Rabin-Karp token match) |
| **Simian** | C-DRY-01 across many formats |
| **Vale** | D-KISS-01/02/03 (prose linting) |
| **markdownlint** | None of these (only structural) |
| **SonarQube** | C-KISS-01/02 (cyclomatic, cognitive complexity) |

**Gap filled by cli-audit-sync:** S family (SSOT code↔doc), C-DRY-01/03/04 (cross-format command/env duplication). No mainstream tool does this.
