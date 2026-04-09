# Gotchas — Multi-Agent Boss Orchestration

> **Rule:** The boss prompt MUST include this section verbatim. Every gotcha comes from a real failure.

---

## G1 — Permission UI blocks the boss (CRITICAL)

**Problem:** Even with `--dangerously-skip-permissions`, the boss (team leader) receives interactive UI prompts to approve worker edits. The boss gets stuck on "Do you want to make this edit?" and waits for a human Enter.

**Cause:** Agent Teams design — the team leader is the gatekeeper for teammate permissions. The `--dangerously-skip-permissions` flag bypasses the boss's own permissions, but NOT those of the workers that bubble up to the leader.

**Fix:** Add `--permission-mode bypassPermissions` to the boss IN ADDITION to `--dangerously-skip-permissions`:
```yaml
claude --dangerously-skip-permissions --permission-mode bypassPermissions --teammate-mode tmux
```

**If forgotten:** The boss will block on every worker edit. You'll have to approve manually with Enter in the boss pane.

---

## G2 — --system-prompt-file does not exist

**Problem:** The `--system-prompt-file` flag does not exist in the Claude Code CLI. The boss will not launch.

**Fix:** Use `--append-system-prompt "$(cat path/to/prompt.md)"` to load a prompt file.

**Alternative:** `--system-prompt "inline content"` but limited for large prompts.

---

## G3 — Claude waits for a first message

**Problem:** Claude Code in interactive mode always waits for a first user message. The boss is launched but does nothing.

**Fix:** Include in the boss prompt a clear instruction to start autonomously. If that is not enough, the user types into the boss pane or we send via tmux:
```bash
tmux send-keys -t {session}:boss "Run the full plan." Enter
```

**DO NOT use `-p` (print mode)** — it disables interactive mode and the teammates' tmux panes will not show up.

---

## G4 — Invisible workers without --teammate-mode tmux

**Problem:** By default, Agent Teams teammates are invisible subprocesses. The user sees nothing in tmux, just the boss.

**Fix:** Always add `--teammate-mode tmux` to the boss:
```yaml
claude --teammate-mode tmux
```

**Permanent alternative:** Set `"teammateMode": "tmux"` in `~/.claude.json`.

---

## G5 — shared-state.md outside the worktree = permission

**Problem:** Workers in worktrees (`grob-wt-hit/`) want to edit `shared-state.md` which lives in the main repo (`grob/.claude/`). This triggers a permission request back to the boss.

**Fix:** Add to `settings.local.json`:
```json
"Edit(//path/to/project/.claude/shared-state.md)"
```

And give the absolute path in the worker prompts:
```
SHARED MEMORY: Read /absolute/path/project/.claude/shared-state.md
```

---

## G6 — Auto-approve Enter too fast

**Problem:** Sending `Enter` keys via tmux send-keys too quickly (< 1s) — the Claude UI ignores them because it hasn't had time to render the prompt.

**Fix:** If you must auto-approve, space them out by at least 3s:
```bash
for i in $(seq 1 N); do tmux send-keys -t session:boss Enter; sleep 3; done
```

**Better:** Use G1 (`--permission-mode bypassPermissions`) so you never see prompts.

---

## G7 — tmuxinator cannot launch Claude in the background

**Problem:** Attempting to launch `claude &` in tmuxinator or to do `bash -c 'claude & sleep...'` breaks the TTY. Claude needs a real terminal.

**Fix:** Let tmuxinator launch Claude normally (foreground in the pane). Do not try to background it.

---

## G8 — Workers receive the wrong prompt

**Problem:** The auto-approve `Enter` keys sent to the boss can land in a worker's input prompt if the tmux focus changed, or the boss kick-off message ends up in a worker pane.

**Fix:** Always target the right pane:
```bash
tmux send-keys -t {session}:0.0 Enter  # boss = always pane 0.0
```

---

## G9 — Merge without CI = catastrophe

**Problem:** The boss merges into develop without waiting for CI. The develop branch is broken.

**Fix:** The boss prompt MUST contain:
```
CI green MANDATORY between every merge.
Loop on: gh run watch
```

---

## G10 — Workers touching the same files

**Problem:** Two workers modify the same file (e.g., `mod.rs`, `Cargo.toml`) → merge conflict guaranteed.

**Fix:**
1. Phase 0: `/cli-audit-tangle` to identify couplings
2. Assign coupled files to the SAME worker
3. "Potential conflicts" section in shared-state.md
4. Workers read "In progress" BEFORE coding

---

## G11 — Too many workers = context window

**Problem:** > 5 workers in parallel. The boss spends too many tokens managing messages/permissions, and the workers step on each other.

**Fix:** Maximum 5 workers. If more than 5 tasks, sequence them across phases.

---

## G12 — on_project_first_start does not exist in tmuxinator

**Problem:** `on_project_first_start` is not a valid tmuxinator hook. The YAML is silently ignored.

**Fix:** Use `on_project_start` (runs on every start). Make the commands idempotent with `2>/dev/null || true`.

---

## G13 — Worktree on an existing branch

**Problem:** `git worktree add ../wt -b feat/x` fails if the branch already exists.

**Fix:** Make it idempotent:
```bash
git worktree add ../wt -b feat/x 2>/dev/null || true
```

---

## G18 — brew upgrade kills workers mid-flight

**Problem:** Agent Teams resolves the `claude` symlink to an absolute path at spawn time (e.g., `Caskroom/claude-code/2.1.84/claude`). If `brew upgrade claude-code` runs during the session, the path points to a deleted version. New workers crash with "No such file or directory".

**Fix:** NEVER run `brew upgrade claude-code` during a multi-agent session. If an upgrade happened: restart the boss (`tmuxinator stop && tmuxinator start`).

**Detection:** If a worker dies with `env: ... No such file or directory`, check `ls /home/linuxbrew/.linuxbrew/Caskroom/claude-code/` — if the spawn-time version is gone, that's it.

---

## G17 — The Chef forgets to update the roadmap

**Problem:** The Chef codes, merges, writes the report, but does not update the project's documentation (roadmap, features, design docs). The next sprint starts with stale docs.

**Fix:** In the Chef's shutdown protocol, MANDATORY step before the final report:
1. Update the roadmap (mark DONE, add next steps)
2. Update the features doc (new capabilities)
3. Update the design docs (statuses PROPOSED → DONE)
4. If Obsidian: update the vault's .md files

**The final report cannot be produced before the docs are up to date.**

---

## G16 — Blind auto-approve = zero safety net

**Problem:** To unblock the boss waiting on worker permissions, people spam Enter in a loop. That approves everything without verification — a worker could edit a sensitive file, push malicious code, or touch files outside its scope.

**Fix:** NEVER blind-auto-approve. The Sous-Chef must read the diff before approving. Sous-Chef rules:
- **APPROVE**: edit within the worker's scope (its assigned files + shared-state.md)
- **DENY**: edit outside scope (another worker's file, .env, credentials, ci.yml without a request)
- **DENY**: removal of tests or security checks
- **DENY**: change to Cargo.toml deps without justification
- **ESCALATE**: if in doubt, ask the Chef

**If no Sous-Chef** (2-tier session): the user IS the Sous-Chef. They watch the boss pane and approve/deny manually.

---

## G15 — Parallel PRs on the same file = conflict

**Problem:** The chef opens N parallel PRs that touch the same file (e.g., ci.yml). The first merges fine, the rest have conflicts.

**Fix:** If several tasks touch the same file, sequence them in the PERT (not parallel). Or better: merge locally in order, test, then push a single PR.

**Rule for the sous-chef:** Before merging a PR, check if another open PR touches the same file. If so, merge the first, rebase the second, then merge.

---

## G14 — Read access to external repos

**Problem:** A worker needs to read another repo (e.g., sokolsky) but does not have the permissions.

**Fix:** Add to `settings.local.json`:
```json
"Read(//path/to/other/repo/**)"
```

---

## G19 — Agent Teams not enabled = no TeamCreate/SendMessage

**Problem:** The skill generates a prompt with TeamCreate but Agent Teams is not enabled. The boss does not have the tools.

**Automatic fix in the skill (Phase 0.3):**
1. Check `~/.claude/settings.json` for `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS`
2. Check `~/.claude.json` for `teammateMode`
3. If missing: add them automatically BEFORE generating the tmuxinator
4. Check `skipDangerousModePermissionPrompt` in `~/.claude/settings.json`

```bash
# Automatic check + fix
python3 -c "
import json, os
p = os.path.expanduser('~/.claude/settings.json')
d = json.load(open(p)) if os.path.exists(p) else {}
d.setdefault('env', {})['CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS'] = '1'
d['skipDangerousModePermissionPrompt'] = True
json.dump(d, open(p, 'w'), indent=2)
"
```

---

## G20 — The Chef has to send the first message manually

**Problem:** Claude Code in interactive mode always waits for a message. tmuxinator launches Claude, but nothing happens. The user has to open the pane and type "Run the plan".

**Fix in tmuxinator**: add an `on_project_start` that sends the first message after a delay:

```yaml
on_project_start:
  # ... worktree setup ...
  # Auto-kick the chef after 12s (Claude boot time)
  - sleep 12 && tmux send-keys -t {session}:conductor "Run the full service. Autonomous pilot." Enter &
```

The trailing `&` launches in the background — tmuxinator does not block on it.

---

## G21 — Idle boss thinks workers are running

**Problem:** The workers are dead (G18, crash, timeout) but the boss shows `Idle · teammates running`. It waits for reports that will never come.

**Fix in the Chef prompt**: add a watchdog:

```
WATCHDOG: If you've been idle for more than 10 minutes and no worker has sent you a message, check whether they are still alive. Try to ping them:
  SendMessage { to: "worker-X", summary: "ping", message: "Are you alive? Answer." }
If no answer within 30s: the worker is dead. Do the work yourself or restart.
```

---

## G22 — Obsidian docs not in permissions scope

**Problem:** The Chef must update the Obsidian docs (roadmap, features) but the Obsidian vault lives outside the project. Edits are blocked by permissions.

**Fix in settings.local.json**: add the Obsidian paths:

```json
"Edit(//{obsidian_vault_path}/**)",
"Read(//{obsidian_vault_path}/**)"
```

---

## G23 — No fallback when workers crash

**Problem:** If all workers are dead (G18) and you cannot restart tmuxinator, the sprint is stuck.

**Fix in the Chef prompt**: an explicit Plan B:

```
IF THE WORKERS ARE DEAD:
1. Do NOT try to re-spawn (same error)
2. Do the work YOURSELF directly in this session
3. Use the existing worktrees (cd /path/to/worktree)
4. One job at a time (no parallelism) but keep moving
5. Tell the user: "Workers dead, continuing solo"
```
