# CLAUDE.md Template — Autophagy Instructions

> **When to read:** During Bootstrap Mode (Step 4) to generate the autophagy instructions for the project's CLAUDE.md.

---

## Template

```markdown
## Semantic Drift Surveillance (Autophagie)

This project uses CONTRACTS.md to record the intended behavior of critical functions.
Before any commit or code review touching a contracted function, scan for drift:

### Scan protocol

1. Read `CONTRACTS.md`
2. For each function that was modified (or that you're about to modify):
   - Check the implementation against ALL listed invariants
   - Search for red flag patterns listed in the contract
   - Verify that no conditional branch treats uniformly-expected parameters differently
   - Compare actual behavior against the stated intention
3. If drift detected:
   - **Name it precisely** ("mask treats field 'b' differently from field 'a' at line 47")
   - **Classify it**: louper (bug) vs evolution (intentional change) vs ambiguity (vague contract)
   - **If louper**: propose the minimal fix, do NOT auto-apply
   - **If evolution**: propose a CONTRACTS.md update AND the code change
   - **If ambiguity**: ask for clarification before proceeding
4. Never assume "it compiles, so it's correct" — the question is: **does it do what the intention says?**

### When to update CONTRACTS.md

- New function added to the public API → add a contract
- Existing behavior intentionally changed → update the contract FIRST, then change the code
- Drift found and fixed → add to "Known drifts" with date and status
- Contract found to be vague → refine invariants to be more precise

### What NOT to do

- Do not update CONTRACTS.md to match a bug — fix the code
- Do not auto-fix drift without asking — name it, classify it, propose
- Do not skip the scan because "it's a small change" — small changes cause the biggest drifts
```
