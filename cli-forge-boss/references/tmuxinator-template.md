# Template — Tmuxinator Config (3-Tier Chain)

Generate at `~/.config/tmuxinator/{session_name}.yml`.

## Embedded gotchas

- G1: `--permission-mode bypassPermissions` on the conductor
- G2: `--append-system-prompt "$(cat ...)"` not `--system-prompt-file`
- G4: `--teammate-mode tmux` to see Guardian + Workers
- G7: No `&` or background on claude
- G12: `on_project_first_start` does NOT exist, use `on_project_start`
- G13: `2>/dev/null || true` on git worktree add
- G24: The ccheck window is MANDATORY — without it, the conductor blocks on every worker permission

---

```yaml
name: {session_name}
root: {project_path}

# --- Worktrees (idempotent — G13) ---
on_project_start:
{worktree_setup_commands}
  # e.g.: - cd {project_path} && git worktree add ../project-wt-feature -b feat/feature 2>/dev/null || true

on_project_stop:
{worktree_cleanup_commands}
  # e.g.: - cd {project_path} && git worktree remove ../project-wt-feature 2>/dev/null || true

windows:
  # --- CONDUCTOR: creates the team, spawns guardian + workers ---
  # GOTCHAS: G1 (bypassPermissions), G2 (append-system-prompt), G4 (teammate-mode tmux)
  - conductor:
      root: {project_path}
      panes:
        - claude --dangerously-skip-permissions --permission-mode bypassPermissions --teammate-mode tmux --append-system-prompt "$(cat {project_path}/.claude/prompts/conductor-{session_name}.md)"

  # --- GATE: merge, CI, release (pure shell, no claude) ---
  # The Guardian sends commands here via tmux send-keys
  - gate:
      root: {project_path}
      panes:
        - echo "=== GATE — merge, rebase, test, build, push, CI ==="

  # --- CCHECK: permission auto-approver (the "contre-chef") ---
  # MANDATORY window. Runs a Claude instance that watches the conductor pane
  # and auto-approves permissions in normal zones, skips sensitive zones.
  # Without this window, the conductor blocks on every worker edit. (G24)
  - ccheck:
      root: {project_path}
      panes:
        - claude --dangerously-skip-permissions --permission-mode bypassPermissions --append-system-prompt "$(cat {project_path}/.claude/prompts/ccheck-{session_name}.md)"
```

## Architecture in tmux

```
Window 1: conductor
  ┌──────────────────┬─────────────┬──────────────┬──────────────┐
  │ Conductor        │ Guardian    │ Worker 1     │ Worker 2     │
  │ (plans)          │ (validates) │ (codes)      │ (codes)      │
  │                  │             │              │              │
  │ SendMessage →    │ gates,      │ commit,      │ commit,      │
  │ receives reports │ merge,      │ SendMessage  │ SendMessage  │
  │                  │ CI          │ → Guardian   │ → Guardian   │
  └──────────────────┴─────────────┴──────────────┴──────────────┘

Window 2: gate
  ┌────────────────────────────────────────────────────────────────┐
  │ Pure shell — git commands sent by the Guardian                 │
  │ git merge, cargo test, git push, gh run watch                  │
  └────────────────────────────────────────────────────────────────┘

Window 3: ccheck (MANDATORY — the contre-chef)
  ┌────────────────────────────────────────────────────────────────┐
  │ Claude instance watching the conductor pane                    │
  │ Auto-approves normal zones, skips sensitive zones              │
  │ Logs every decision for traceability                           │
  └────────────────────────────────────────────────────────────────┘
```

## Notes

- **3 tmuxinator windows**: `conductor`, `gate`, and `ccheck`
- The Guardian and Workers are spawned by the Conductor via Agent Teams
- They appear in tmux panes thanks to `--teammate-mode tmux`
- The Guardian uses `mode: "bypassPermissions"` — zero UI blocking (G1)
- The ccheck watches the conductor pane and handles permissions (G24)
- Maximum 5 workers + 1 guardian = 6 teammates + conductor = 7 panes in window 1
