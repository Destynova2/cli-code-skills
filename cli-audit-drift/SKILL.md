---
name: cli-audit-drift
description: >
  Detect silent semantic drift between intended behavior (CONTRACTS.md) and actual implementation.
  Scans code against functional contracts, invariants, and known drift history to catch behavioral
  changes that compile and run but violate the original intention. Use when reviewing code changes,
  before commits, auditing behavioral conformity, or saying 'check drift', 'contract check',
  'intention vs implementation', 'semantic drift', 'is this still correct', 'does this match the spec',
  'behavioral regression', 'silent bug', 'autophagy scan'. Also triggers on 'CONTRACTS.md',
  'invariant check', 'intention audit', 'contract violation'.
argument-hint: "[file-or-directory-or-function-name]"
context: fork
agent: general-purpose
allowed-tools:
  - Read
  - Write
  - Grep
  - Glob
  - Bash
  - Agent
---

> **Optimization:** This skill uses on-demand loading. Heavy content lives in `references/` and is loaded only when needed.

> **Language rule:** Detect the project's primary language (from README, comments, docs, commit messages). Output your report in that language. If the project is bilingual, ask the user which language to use before proceeding.

# Audit Drift — Semantic Conformity Scanner

Detect silent behavioral drift between what code **was supposed to do** (intention) and what it **actually does** (implementation). The most dangerous bugs are not crashes — they are functions that still run, still return values, but no longer honor their original contract.

> "Le corbeau de Nouvelle-Calédonie ne subit pas son environnement — il identifie l'inadéquation entre ce qu'il voulait faire et ce qui s'est passé. Pas une erreur bruyante. Un résultat qui ne correspond pas à l'intention."

## Core Principle — Two biological mechanisms

| Organism | Mechanism | What it does in testing |
|----------|-----------|------------------------|
| **Corbeau de Nouvelle-Calédonie** | Gap detection between intention and result | Compares CONTRACTS.md (intention) against code (result). Names the gap precisely. Asks if intentional or accidental |
| **Autophagie cellulaire** | Continuous conformity surveillance | Scans all contracted functions, isolates non-conforming code, proposes minimal correction or contract update |

## Input

`$ARGUMENTS` is the target to scan:

- **Function name** (`mask`, `calculate_price`): scan that specific function against its contract
- **File or directory** (`src/core/`, `lib.rs`): scan all contracted functions found in that scope
- **Empty**: scan the entire project against all contracts in CONTRACTS.md

## Workflow

### Step 0 — Locate or bootstrap CONTRACTS.md + CLAUDE.md check

1. Search for `CONTRACTS.md` at project root, then `docs/CONTRACTS.md`, then `contracts/`
2. If found: read it, proceed to Step 0b
3. If NOT found: switch to **Bootstrap Mode** (see below)

**Step 0b — Verify CLAUDE.md autophagy instructions:**

1. Search for `CLAUDE.md` (or `.claude/settings.json` project instructions) at project root
2. If CLAUDE.md exists, check if it contains autophagy/drift surveillance instructions
3. If **missing autophagy section**: read `references/claude-md-template.md` and append the section to the report as a recommended CLAUDE.md update
4. If **autophagy section exists but outdated** (references functions no longer in CONTRACTS.md, or missing new contracted functions): propose an update

### Step 1 — Parse contracts

For each contracted function/endpoint, extract:
- **Intention**: what it should do (natural language)
- **Invariants**: properties that must always hold (formal-ish)
- **Known drifts**: historical drift log with dates
- **Red flags**: specific code patterns that indicate drift

Read `references/invariant-patterns.md` for common invariant categories.

### Step 2 — Locate implementations

For each contracted function:
1. Find its definition (grep for function signature, method, endpoint)
2. Read the full implementation
3. If the function calls other contracted functions, note the chain

### Step 3 — Scan for drift (Corbeau)

For each contracted function, check:

**Invariant violations:**
- Does the implementation honor every listed invariant?
- Are there conditional branches that break uniformity invariants?
- Are there side effects not declared in the contract?

**Red flag detection:**
- Does the code contain any pattern listed in "Red flags"?
- Are there special cases for values that should be treated uniformly?

**Behavioral divergence:**
- Does the function's actual behavior match the stated intention?
- Could a reasonable input produce output that violates the intention?
- Has a recent change (git diff) introduced a divergence?

Read `references/drift-detection-rules.md` for detailed detection heuristics.

### Step 4 — Classify findings (Autophagie)

For each detected drift:

| Classification | Meaning | Action |
|---------------|---------|--------|
| **Louper** | Unintentional deviation from contract | Propose minimal fix |
| **Evolution** | Intentional change, contract not updated | Propose contract update + alert stakeholders |
| **Ambiguity** | Contract is vague, implementation chose one interpretation | Propose contract clarification |
| **Stale contract** | Contract references code that no longer exists | Propose contract cleanup |

### Step 5 — Generate report

Use the output format below.

### Step 6 — Update drift history

If drifts were found, propose additions to the "Historique des dérives connues" section of each affected contract.

## Bootstrap Mode

When no CONTRACTS.md exists, the skill switches from audit to generation:

1. **Discover critical functions**: Scan the codebase for functions that are:
   - Public API endpoints
   - Core business logic (domain layer)
   - Functions with complex invariants (state machines, calculators, validators)
   - Functions that have had bugs (check git log for fix commits)

2. **Infer intentions**: For each function, read:
   - The function's docstring/comments
   - Its test cases (what do tests assert?)
   - Its git history (what was it supposed to fix/add?)
   - Its callers (what do consumers expect?)

3. **Generate CONTRACTS.md**: Read `references/contracts-template.md` for the template. Generate contracts for the top 5-10 critical functions.

4. **Propose CLAUDE.md additions**: Read `references/claude-md-template.md` for the autophagy instructions to add to the project's CLAUDE.md.

5. **Output**: deliver CONTRACTS.md + CLAUDE.md snippet, explain how to use them.

## Output Format

```markdown
# Semantic Drift Audit — {project-name}

**Date:** {date}
**Scope:** {what was scanned}
**Contracts loaded:** {N} functions from CONTRACTS.md
**Drifts detected:** {N} ({loupers} loupers, {evolutions} evolutions, {ambiguities} ambiguities)

## Drift Report

### {function-name} — {drift-classification}

**Contract says:** {quoted intention from CONTRACTS.md}
**Implementation does:** {observed behavior}
**Gap:** {precise description of the divergence}
**Evidence:** `{file}:{line}` — {code snippet showing the drift}
**Invariant violated:** {which invariant, if applicable}

**Recommendation:**
- {action: fix / update contract / clarify / ask PO}
- {minimal code change if louper}

---
(repeat for each drift)

## Contract Health Summary

| Function | Status | Last verified | Drifts found |
|----------|--------|--------------|-------------|
| mask(data, fields) | DRIFT | {date} | 1 louper |
| calculate_price() | CLEAN | {date} | 0 |
| ... | | | |

## Proposed CONTRACTS.md Updates

(diff-style additions to the drift history section)

## CLAUDE.md Status

**Autophagy instructions:** {PRESENT / MISSING / OUTDATED}
{if MISSING or OUTDATED: proposed additions below}

### Proposed CLAUDE.md additions

(copy-pasteable block with autophagy instructions)
```

## Rules for the Scanner

1. **Never auto-fix silently.** Name the gap, classify it, propose the fix — but always ask. The corbeau identifies inadequacy before acting.
2. **Contract is truth until proven intentionally outdated.** If code disagrees with the contract, the code is suspect — not the contract. Only the human decides to update the contract.
3. **Minimal fixes only.** Don't refactor. Don't improve. Fix the exact drift, nothing more.
4. **Check recent changes first.** `git diff` and `git log` on contracted functions are the first place drift appears.
5. **Invariants > intentions.** A vague intention is hard to audit. A formal invariant is machine-checkable. Prioritize invariant violations.
6. **Chain awareness.** If `process()` calls `mask()` and `mask()` has drifted, `process()` is also affected. Report the chain.
7. **Gotchas** — read `../../gotchas.md` before producing output to avoid known mistakes.

## Integration with other cli-* skills

| Skill | Relationship |
|-------|-------------|
| `cli-audit-test` | D13 audits whether drift detection **tools** exist. cli-audit-drift **is** the drift detection |
| `cli-audit-code` | Scores code quality. cli-audit-drift checks **behavioral conformity** to intentions |
| `cli-audit-sync` | Checks doc-code coherence. cli-audit-drift checks **contract-code coherence** |
| `cli-forge-doc` | Can generate documentation. cli-audit-drift generates **CONTRACTS.md** (a different artifact) |
| `cli-cycle` | Should call cli-audit-drift as part of the full project review |

## What this skill does NOT do

- **Does not run tests** — it reads code and contracts
- **Does not replace tests** — it complements them (tests check examples, contracts check intentions)
- **Does not auto-fix** — it proposes, the human decides
- **Does not generate test code** — it generates contract documents
- **Does not check compilation** — it checks semantic conformity
