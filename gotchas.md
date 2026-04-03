# Gotchas — Skill Execution Lessons Learned

> **Rule:** Every skill MUST read this file before producing output.
> When you make a mistake related to a skill, add it here immediately.
> Format: `[date] [skill] — what went wrong → what to do instead`

---

## Architecture & Design

- [2026-03-26] cli-forge-hld/lld — STRIDE threat model was entirely in LLD. STRIDE has two levels: system-level trust boundaries (HLD) and component-level threats like SQLi/IDOR (LLD). Always split.
- [2026-03-26] cli-forge-hld/lld — Domain Model was entirely in LLD. Strategic DDD (bounded contexts, context maps) = HLD. Tactical DDD (aggregates, entities, VOs) = LLD. Always split.
- [2026-03-26] cli-forge-hld/lld — "Mathematical" ≠ "low-level". Scalability laws (Little, Amdahl) in capacity estimation = HLD. Connection pool config using those laws = LLD. Same formula, different scope.
- [2026-03-26] cli-forge-hld/lld — All 18 HLD sections were mandatory. Documents became bloated for small projects. Use mitosis: tier S/M/L/XL determines which sections to include.

## Scoring & Output

- [2026-03-26] cli-forge-hld/lld — Review checklists listed everything as mandatory, contradicting the mitosis principle. Checklists must be tiered too.

## CI/CD

- [2026-03-27] cli-forge-pipeline — Bumped `cosign-installer@v3` → `@v4` assuming the major floating tag existed. It didn't (only `v4.0.0`, `v4.1.1` existed). Always verify tag existence via `gh api repos/{owner}/{repo}/git/refs/tags/{tag}` before recommending a bump. If no floating major tag, pin to latest patch (e.g., `@v4.1.1`).

## Shell & Infra

- [2026-04-03] cli-audit-shell — `set -euo pipefail` changes bash semantics fundamentally: `set -e` disables itself inside `if`/`while`/`||`/`&&` conditions but NOT in standalone statements. A `command_a; command_b` pair where `command_a` is standalone will exit before reaching `command_b`. Don't report dead fallback if the command is inside a condition.
- [2026-04-03] cli-forge-infra — Don't recommend kustomize for manifests with 1-3 variable substitutions. The tooling ladder (sed < yq < kustomize < Helm) must match the complexity. Over-engineering is a finding, not a solution.
- [2026-04-03] cli-audit-shell — `local var="$(cmd)"` masks `cmd`'s exit code because `local` returns 0. But this is ONLY a problem if you check `$?` immediately after. If you use `set -e`, the exit code of `cmd` IS propagated (bash 4.4+). Check the bash version context before flagging.

## General

- [2026-03-26] all skills — SKILL.md files grew to 700-800 lines. Everything loaded into context = token waste. Split: lean SKILL.md (~150 lines workflow) + references/ (loaded on-demand).
