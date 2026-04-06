# Template — Tmuxinator Config (3-Tier Chain)

Generer dans `~/.config/tmuxinator/{session_name}.yml`.

## Gotchas integres

- G1: `--permission-mode bypassPermissions` sur le conductor
- G2: `--append-system-prompt "$(cat ...)"` pas `--system-prompt-file`
- G4: `--teammate-mode tmux` pour voir Guardian + Workers
- G7: Pas de `&` ou background sur claude
- G12: `on_project_first_start` n'existe PAS, utiliser `on_project_start`
- G13: `2>/dev/null || true` sur les git worktree add

---

```yaml
name: {session_name}
root: {project_path}

# --- Worktrees (idempotent — G13) ---
on_project_start:
{worktree_setup_commands}
  # ex: - cd {project_path} && git worktree add ../project-wt-feature -b feat/feature 2>/dev/null || true

on_project_stop:
{worktree_cleanup_commands}
  # ex: - cd {project_path} && git worktree remove ../project-wt-feature 2>/dev/null || true

windows:
  # --- CONDUCTOR : cree le team, spawn guardian + workers ---
  # GOTCHAS: G1 (bypassPermissions), G2 (append-system-prompt), G4 (teammate-mode tmux)
  - conductor:
      root: {project_path}
      panes:
        - claude --dangerously-skip-permissions --permission-mode bypassPermissions --teammate-mode tmux --append-system-prompt "$(cat {project_path}/.claude/prompts/conductor-{session_name}.md)"

  # --- GATE : merge, CI, release (shell pur, pas de claude) ---
  # Le Guardian envoie des commandes ici via tmux send-keys
  - gate:
      root: {project_path}
      panes:
        - echo "=== GATE — merge, rebase, test, build, push, CI ==="
```

## Architecture dans tmux

```
Fenetre 1: conductor
  ┌──────────────────┬─────────────┬──────────────┬──────────────┐
  │ Conductor        │ Guardian    │ Worker 1     │ Worker 2     │
  │ (planifie)       │ (valide)    │ (code)       │ (code)       │
  │                  │             │              │              │
  │ SendMessage →    │ gates,      │ commit,      │ commit,      │
  │ recoit rapports  │ merge,      │ SendMessage  │ SendMessage  │
  │                  │ CI          │ → Guardian   │ → Guardian   │
  └──────────────────┴─────────────┴──────────────┴──────────────┘

Fenetre 2: gate
  ┌────────────────────────────────────────────────────────────────┐
  │ Shell pur — commandes git envoyees par le Guardian             │
  │ git merge, cargo test, git push, gh run watch                  │
  └────────────────────────────────────────────────────────────────┘
```

## Notes

- Seules 2 fenetres tmuxinator : `conductor` et `gate`
- Le Guardian et les Workers sont spawn par le Conductor via Agent Teams
- Ils apparaissent dans des panes tmux grace a `--teammate-mode tmux`
- Le Guardian utilise `mode: "bypassPermissions"` — zero blocage UI (G1)
- Maximum 5 workers + 1 guardian = 6 teammates + conductor = 7 panes
