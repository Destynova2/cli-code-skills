# shared/

Cross-skill reference files — the common denominators that several `cli-*` skills point to instead of each re-deriving them.

Path convention:

- From a root `cli-*/SKILL.md`, reference shared files as `../shared/<file>.md` and `../gotchas.md`.
- From a `cli-*/references/*.md` file, reference shared files as `../../shared/<file>.md` and `../../gotchas.md`.

| File | What it canonicalizes | Consumed by |
|---|---|---|
| `recon.md` | The "read the project first" brief (what/who/problem/headline/differentiator/golden-path) | readme, prez, demo |
| `gate-ladder.md` | The T0-T4/M0 progressive proof-gate semantics | resilience, oci-rootless, pipeline, audit-test, demo, perf (T4 stress) |
| `determinism.md` | Reproducibility & idempotence toolkit (seed, clock, env, content-hash) | demo, pipeline, resilience, wizard, oci-rootless, perf (reproducible baselines) |
| `result-schema.md` | Machine-readable result envelope `.claude/<skill>.json` | cli-cycle (aggregates), all audit/forge skills (emit) |
| `optimization-card.md` | Canonical optimization hypothesis format: evidence, invariants, validation, risk, owner | audit-xray, perf, audit-drift, audit-test, pipeline |
| `metaphors.md` | Catalog of shared vs. signature metaphors | all biomimetic skills; cli-cycle (consistent vocabulary) |
| `escalation-ladder.md` | 5-rung CLI → file → private API → IPC → GUI ladder for "no headless path exists" | perf, infra, audit-wizard, forge-chef, oci-rootless |
| `cli-ergonomics.md` | The 4 Laws (ask once / defaults / recap / config-as-code) and surface mapping (CLI / TUI / wizard / CI) | audit-wizard, audit-shell, forge-chef, forge-infra |
| `triage.md` | Phoenix 3-2-1 tiers, GRADE confidence, multi-method triangulation | cli-cycle (full Phoenix), forge-resilience (Step 9), all audit-* skills (finding tier × confidence) |
| `tiering.md` | Mitosis S/M/L/XL — scale depth of output to size of input | forge-chef (brigade size), audit-tangle (analysis depth), forge-hld (doc sections), audit-sync (audit layers), forge-pipeline (workflow fission) |
| `done-gate.md` | 3-phase definition-of-done (pre-conditions → during → post-verification) | forge-perf (GATE), forge-resilience (Step 8 score), forge-pipeline (mutation testing), forge-oci-rootless (Step 6 proof), forge-demo (Step 6 quality gate), forge-chef (Phase 3 hard gates), forge-doc (DCI completeness) |
| `data-invariants.md` | Database sources of truth, L0-L3 criticality, invariants, conservation, interleavings, proof ladder, runtime uncertainty | audit-data, forge-data, audit-drift, audit-test, audit-review, forge-lld |
| `data-invariant-catalog.md` | Detailed question catalog for discovering persistent business invariants | audit-data, forge-data |
| `postgres-sqlx-patterns.md` | PostgreSQL and SQLx implementation patterns | audit-data, forge-data |
| `ai-tells.md` | Typographic, lexical (FR + EN), structural and process markers of AI-written copy, with rewrite directions; the pass every outgoing message goes through | forge-plume, forge-nego, forge-influence |

These ship to `~/.claude/skills/shared/` alongside `~/.claude/skills/gotchas.md`.
