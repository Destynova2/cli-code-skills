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
- [2026-04-06] cli-forge-pipeline — Missed `cargo mutants` as a fan-out candidate. Any job running the same tool on N files sequentially (`--file A --file B --file C`) is a fan-out/shard opportunity. The fourmis légionnaires pattern (#3) applies to ALL slow sequential jobs, not just tests. Checklist: grep for `--file`, `--shard`, `--partition` in CI YAML — if a job lists multiple targets inline, propose sharding into a matrix.

## Shell & Infra

- [2026-04-03] cli-audit-shell — `set -euo pipefail` changes bash semantics fundamentally: `set -e` disables itself inside `if`/`while`/`||`/`&&` conditions but NOT in standalone statements. A `command_a; command_b` pair where `command_a` is standalone will exit before reaching `command_b`. Don't report dead fallback if the command is inside a condition.
- [2026-04-03] cli-forge-infra — Don't recommend kustomize for manifests with 1-3 variable substitutions. The tooling ladder (sed < yq < kustomize < Helm) must match the complexity. Over-engineering is a finding, not a solution.
- [2026-04-03] cli-audit-shell — `local var="$(cmd)"` masks `cmd`'s exit code because `local` returns 0. But this is ONLY a problem if you check `$?` immediately after. If you use `set -e`, the exit code of `cmd` IS propagated (bash 4.4+). Check the bash version context before flagging.

## Convergence

- [2026-04-03] cli-cycle — En mode convergence autonome, ne JAMAIS auto-corriger les items marqués "décision métier" ou "décision architecture" (CHANGELOG, LICENSE, overlay prod, etc.). Ces items sont DEFERRED et listés pour le user, pas corrigés silencieusement.
- [2026-04-03] cli-cycle — Si une passe introduit PLUS de Tier 3 qu'elle n'en résout, STOP immédiat. C'est un signe de divergence (la correction crée plus de problèmes qu'elle n'en résout). Présenter l'état actuel et laisser le user décider.

## Multi-Agent / Boss

- [2026-04-06] cli-forge-boss — `--dangerously-skip-permissions` does NOT bypass teammate permission prompts in Agent Teams. The team leader still gets interactive UI prompts for worker edits. Must add `--permission-mode bypassPermissions` as well.
- [2026-04-06] cli-forge-boss — `--system-prompt-file` does not exist in Claude Code CLI. Use `--append-system-prompt "$(cat path/to/file.md)"` instead.
- [2026-04-06] cli-forge-boss — `--teammate-mode tmux` is required to see workers in separate tmux panes. Without it, workers are invisible subprocesses.
- [2026-04-06] cli-forge-boss — `on_project_first_start` is not a valid tmuxinator hook. Use `on_project_start` with idempotent commands (`2>/dev/null || true`).
- [2026-04-06] cli-forge-boss — Workers editing shared-state.md (outside their worktree) triggers permission requests to the boss. Add explicit `Edit(//path/.claude/shared-state.md)` in settings.local.json.
- [2026-04-06] cli-forge-boss — Sending Enter via tmux send-keys too fast (< 1s interval) gets ignored by Claude UI. Use 3s minimum spacing if auto-approving.
- [2026-04-06] cli-forge-boss — Claude Code in interactive mode always waits for a first user message. The boss won't start autonomously from system prompt alone — needs a kick message.
- [2026-04-06] cli-forge-boss — Do NOT use `-p` (print mode) for the boss. Print mode disables interactive mode and teammate tmux panes won't appear.

## File extensions

- [2026-04-08] cli-forge-tree / cli-cycle — Ne JAMAIS renommer `.yml` ↔ `.yaml` dans un projet existant. Détecter la convention du projet (compter `*.yml` vs `*.yaml`) et utiliser celle-ci pour les nouveaux fichiers. Exceptions : Kustomize/Helm imposent `.yaml`, GitLab CI impose `.gitlab-ci.yml`, ces extensions sont non-négociables. Pour tout le reste, respecter ce qui existe.

## OPSEC / Stealth

- [2026-04-07] cli-forge-doc — AGENTS.md et CLAUDE.md sont des marqueurs AI visibles dans un repo. Score DREAD 8.8/10 : Information Disclosure (architecture complète, gotchas exploitables) + Repudiation (trace AI dans l'historique). En mode stealth, redistribuer tout le contenu dans CONTRIBUTING.md, docs/explanation/architecture.md, docs/TROUBLESHOOTING.md. Zéro fichier nommé "CLAUDE", "AGENTS", ou "llms".
- [2026-04-07] cli-git-conventional — Le Co-authored-by: Claude est banni par le skill. Mais vérifier aussi l'historique git existant : `git log --grep="Co-authored-by.*Claude" --grep="Co-authored-by.*AI" --grep="Generated by"`. Si trouvé, avertir le user (impossible à supprimer sans force push/rebase interactif).

## General

- [2026-03-26] all skills — SKILL.md files grew to 700-800 lines. Everything loaded into context = token waste. Split: lean SKILL.md (~150 lines workflow) + references/ (loaded on-demand).
