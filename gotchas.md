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

## General

- [2026-03-26] all skills — SKILL.md files grew to 700-800 lines. Everything loaded into context = token waste. Split: lean SKILL.md (~150 lines workflow) + references/ (loaded on-demand).
