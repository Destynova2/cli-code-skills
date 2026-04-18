# Upgrade Path — From cli-forge-chef (Brigade) to cli-forge-quorum (REC-Q)

A working Brigade is a valid stepping stone. Do not throw it away. Migrate incrementally — each step is independently valuable and testable.

## Six incremental steps

### Step 1 — Disarm the Chef bottleneck
**What**: spawn a `contre-chef-inter` (already provided by Brigade at tier ≥ M since G29).
**Effect**: inter-agent permission requests get automated recommendations to the Chef. Chef's approval decision is O(1) (just forward the signature).
**Cost**: one Haiku session (~$0.05/sprint).
**Risk**: low — the Chef still has final say.

### Step 2 — Introduce tickets pré-signés
**What**: at Phase 0, Chef outputs a signed ticket per commis (see `leaderless-execute.md` for schema).
**Effect**: contre-chef-inter verifies against ticket scope instead of shared-state ad hoc. No round-trip to Chef for in-scope edits.
**Cost**: signing infrastructure (Ed25519 key per commis, ~100 lines of code).
**Risk**: medium — requires a key management scheme. Use ephemeral keys per sprint if simpler.

### Step 3 — Split Sous-Chef roles
**What**: rename Brigade's 3 Sous-Chefs:
- `sous-chef-scope` + `sous-chef-secu` → Phase 0 **Plan-Validators** (vote on plan hash)
- `sous-chef-qualite` + (new) `control-tests` + (new) `control-contracts` → Phase 2 **Control-Validators**
- Brigade's `sous-chef-merge` + `maitre-d-hotel` → merge + watchdog (unchanged)

**Effect**: clear phase boundaries. Each validator is specialized for one phase, one question.
**Cost**: refactor 3 Sous-Chef prompts → 5-6 more narrow prompts.
**Risk**: low — prompts are just text, easy to iterate.

### Step 4 — Add per-phase timeouts with view-change
**What**: every phase gets a deadline. On expiry, spawn backup agent with last-known state.
**Effect**: no more 45-min stalls. Worst case: view-change replaces a stuck agent and phase continues.
**Cost**: implement a small watchdog (monitor tool call duration, spawn backup on expiry). ~50 lines of orchestration code.
**Risk**: medium — view-change can fire too eagerly on slow (not stuck) agents; tune timeouts carefully.

**Suggested initial timeouts**:
- Phase 0 (Reflect): 10 min total
- Phase 1 (Execute): 2× PERT-E per task
- Phase 2 (Control per validator): 5 min

### Step 5 — Migrate shared-state to append-only JSONL
**What**: replace `shared-state.md` (edited in place) with `shared-state.jsonl` (append-only + hash chain + signatures).
**Effect**: audit trail, state reconstruction, no merge conflicts on concurrent writes.
**Cost**: adapt commis prompts to emit JSONL entries instead of editing markdown. ~1 day of adjustments.
**Risk**: medium — existing automations reading shared-state.md need to be updated.

### Step 6 — Petri net / BPMN visualization
**What**: generate a workflow diagram (see `petri-cwn-templates.md`) alongside the prompts. Run soundness analysis.
**Effect**: formal check that there are no deadlocks. Optional live dashboard.
**Cost**: ~half-day one-time, then included automatically in `cli-forge-quorum` generation.
**Risk**: low — if the diagram reveals a deadlock, you needed to know anyway.

## Migration status → skill to run

| Your brigade has... | Run skill | Expected outcome |
|---|---|---|
| Just Chef + Sous-Chef + Commis | `cli-forge-chef` | Still fine, keep going |
| Added ccheck | `cli-forge-chef` (v2 with inter) | Good, covers 80% of stalls |
| Added contre-chef-inter + tickets | `cli-forge-chef` or `cli-forge-quorum` | Choice based on next needs |
| Wants per-phase timeouts + view-change | `cli-forge-quorum` | Brigade alone doesn't spec these |
| Needs compliance audit trail | `cli-forge-quorum` | JSONL + sigs + Petri required |
| Wants formal BFT | `cli-forge-quorum` | With HotStuff-lite backend |

## What to keep (shared between Brigade and REC-Q)

These templates live under `cli-forge-chef/references/` and are **imported** by `cli-forge-quorum`:

- `ccheck-prompt-template.md` — UI keyboard intercept, unchanged role
- `contre-chef-inter-prompt-template.md` — inter-agent permission auto-nudger, unchanged role
- `tmuxinator-template.md` — base YAML structure
- `maitre-dhotel.md` — post-merge watchdog
- `gotchas-chef.md` — Claude Code pitfalls (teammate-mode, bypassPermissions, etc.)

Do not duplicate. Use path references: `{cli-forge-chef}/references/ccheck-prompt-template.md`.

## What changes in REC-Q

These concepts are REC-Q-specific (not in Brigade):

- `rec-phases.md` — three distinct phases with signed certificates
- `leaderless-execute.md` — tickets pré-signés + per-commis scopes
- `quorum-sizing.md` — criticality classifier + cost model
- `petri-cwn-templates.md` — formal workflow modeling
- `bft-primitives.md` — consensus library pointers

## Anti-patterns during migration

- **Big bang rewrite** — doesn't work. Each step must be testable independently.
- **Adopt step 6 (Petri) without steps 1-5** — a pretty diagram of a broken process is still broken.
- **Skip step 4 (timeouts) because "the Chef works"** — that's exactly how grob-s5 burned 45 min. Every phase needs a timeout.
- **Use both Brigade and REC-Q for different sprints of the same project** — pick one and stick with it. Mixing creates confusion about which shared-state format is canonical.
- **Port all Brigade features verbatim** — REC-Q explicitly drops the Chef's ad-hoc approval role. That's not a bug to fix.

## Rollback plan

If REC-Q proves over-engineered for a given project, rolling back is straightforward:

1. Keep last-known-good Brigade config (`cli-forge-chef` output) in a branch
2. Switch shared-state.jsonl reader → shared-state.md reader in automations
3. Disable view-change watchdog
4. Merge Plan-Validators back into generic Sous-Chefs

Expect ~2 hours of work to revert. Before migrating, tag the current Brigade state: `git tag brigade-last-known-good`.
