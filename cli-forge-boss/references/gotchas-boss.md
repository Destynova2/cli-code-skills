# Gotchas — Multi-Agent Orchestration

> **Rule:** The Chef prompt MUST include this section verbatim. Every gotcha comes from a real failure.
>
> **Terminology:** "Chef" = the planning agent (Brigade role). "conductor" = the tmux window/pane name. They are the same agent. Never use "boss" in prompts or configs — it's only the skill name (`cli-forge-boss`).

---

## G1 — Permission UI blocks the Chef (CRITICAL)

**Problem:** Even with `--dangerously-skip-permissions`, the Chef (team leader) receives interactive UI prompts to approve commis edits. The Chef gets stuck on "Do you want to make this edit?" and waits for a human Enter.

**Cause:** Agent Teams design — the team leader is the gatekeeper for teammate permissions. The `--dangerously-skip-permissions` flag bypasses the Chef's own permissions, but NOT those of the commis that bubble up to the leader.

**Fix:** Add `--permission-mode bypassPermissions` to the Chef IN ADDITION to `--dangerously-skip-permissions`:
```yaml
claude --dangerously-skip-permissions --permission-mode bypassPermissions --teammate-mode tmux
```

**If forgotten:** The Chef will block on every commis edit. You'll have to approve manually with Enter in the Chef pane.

---

## G2 — --system-prompt-file does not exist

**Problem:** The `--system-prompt-file` flag does not exist in the Claude Code CLI. The Chef will not launch.

**Fix:** Use `--append-system-prompt "$(cat path/to/prompt.md)"` to load a prompt file.

**Alternative:** `--system-prompt "inline content"` but limited for large prompts.

---

## G3 — Claude waits for a first message

**Problem:** Claude Code in interactive mode always waits for a first user message. The Chef is launched but does nothing.

**Fix:** Include in the Chef prompt a clear instruction to start autonomously. If that is not enough, the user types into the Chef pane or we send via tmux:
```bash
tmux send-keys -t {session}:chef "Run the full plan." Enter
```

**DO NOT use `-p` (print mode)** — it disables interactive mode and the teammates' tmux panes will not show up.

---

## G4 — Invisible commis without --teammate-mode tmux

**Problem:** By default, Agent Teams teammates are invisible subprocesses. The user sees nothing in tmux, just the Chef.

**Fix:** Always add `--teammate-mode tmux` to the Chef:
```yaml
claude --teammate-mode tmux
```

**Permanent alternative:** Set `"teammateMode": "tmux"` in `~/.claude.json`.

---

## G5 — shared-state.md outside the worktree = permission

**Problem:** Commis in worktrees (`grob-wt-hit/`) want to edit `shared-state.md` which lives in the main repo (`grob/.claude/`). This triggers a permission request back to the Chef.

**Fix:** Add to `settings.local.json`:
```json
"Edit(//path/to/project/.claude/shared-state.md)"
```

And give the absolute path in the commis prompts:
```
SHARED MEMORY: Read /absolute/path/project/.claude/shared-state.md
```

---

## G6 — Auto-approve Enter too fast

**Problem:** Sending `Enter` keys via tmux send-keys too quickly (< 1s) — the Claude UI ignores them because it hasn't had time to render the prompt.

**Fix:** If you must auto-approve, space them out by at least 3s:
```bash
for i in $(seq 1 N); do tmux send-keys -t session:chef Enter; sleep 3; done
```

**Better:** Use the ccheck window (G24) which handles this with proper delays.

---

## G7 — tmuxinator cannot launch Claude in the background

**Problem:** Attempting to launch `claude &` in tmuxinator or to do `bash -c 'claude & sleep...'` breaks the TTY. Claude needs a real terminal.

**Fix:** Let tmuxinator launch Claude normally (foreground in the pane). Do not try to background it.

---

## G8 — Commis receive the wrong prompt

**Problem:** The auto-approve `Enter` keys sent to the Chef can land in a worker's input prompt if the tmux focus changed, or the Chef kick-off message ends up in a worker pane.

**Fix:** Always target the right pane:
```bash
tmux send-keys -t {session}:chef.0 Enter  # chef = always pane 0.0
```

---

## G9 — Merge without CI = catastrophe

**Problem:** The Chef merges into develop without waiting for CI. The develop branch is broken.

**Fix:** The Chef prompt MUST contain:
```
CI green MANDATORY between every merge.
Loop on: gh run watch
```

---

## G10 — Commis touching the same files

**Problem:** Two commis modify the same file (e.g., `mod.rs`, `Cargo.toml`) → merge conflict guaranteed.

**Fix:**
1. Phase 0: `/cli-audit-tangle` to identify couplings
2. Assign coupled files to the SAME worker
3. "Potential conflicts" section in shared-state.md
4. Commis read "In progress" BEFORE coding

---

## G11 — Too many commis = context window

**Problem:** > 5 commis in parallel. The Chef spends too many tokens managing messages/permissions, and the commis step on each other.

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

## G14 — Read access to external repos

**Problem:** A worker needs to read another repo (e.g., sokolsky) but does not have the permissions.

**Fix:** Add to `settings.local.json`:
```json
"Read(//path/to/other/repo/**)"
```

---

## G15 — Parallel PRs on the same file = conflict

**Problem:** The Chef opens N parallel PRs that touch the same file (e.g., ci.yml). The first merges fine, the rest have conflicts.

**Fix:** If several tasks touch the same file, sequence them in the PERT (not parallel). Or better: merge locally in order, test, then push a single PR.

**Rule for the Sous-Chef:** Before merging a PR, check if another open PR touches the same file. If so, merge the first, rebase the second, then merge.

---

## G16 — Blind auto-approve = zero safety net

**Problem:** To unblock the Chef waiting on worker permissions, people spam Enter in a loop. That approves everything without verification — a commis could edit a sensitive file, push malicious code, or touch files outside its scope.

**Fix:** NEVER blind-auto-approve. The ccheck (G24) reads the diff before approving. Ccheck rules:
- **APPROVE**: edit within the commis's scope (its assigned files + shared-state.md)
- **DENY**: edit outside scope (another commis's file, .env, credentials, ci.yml without a request)
- **DENY**: removal of tests or security checks
- **DENY**: change to Cargo.toml deps without justification
- **ESCALATE**: if in doubt, skip and let the user decide

**If no ccheck** (minimal 2-tier session): the user IS the ccheck. They watch the Chef pane and approve/deny manually.

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

## G18 — brew upgrade kills commis mid-flight

**Problem:** Agent Teams resolves the `claude` symlink to an absolute path at spawn time (e.g., `Caskroom/claude-code/2.1.84/claude`). If `brew upgrade claude-code` runs during the session, the path points to a deleted version. New commis crash with "No such file or directory".

**Fix:** NEVER run `brew upgrade claude-code` during a multi-agent session. If an upgrade happened: restart the session (`tmuxinator stop && tmuxinator start`).

**Detection:** If a worker dies with `env: ... No such file or directory`, check `ls /home/linuxbrew/.linuxbrew/Caskroom/claude-code/` — if the spawn-time version is gone, that's it.

---

## G19 — Agent Teams not enabled = no TeamCreate/SendMessage

**Problem:** The skill generates a prompt with TeamCreate but Agent Teams is not enabled. The Chef does not have the tools.

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
  # Auto-kick the Chef after 12s (Claude boot time)
  - sleep 12 && tmux send-keys -t {session}:chef "Run the full service. Autonomous pilot." Enter &
```

The trailing `&` launches in the background — tmuxinator does not block on it.

---

## G21 — Idle Chef thinks commis are running

**Problem:** The commis are dead (G18, crash, timeout) but the Chef shows `Idle · teammates running`. It waits for reports that will never come.

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

## G23 — No fallback when commis crash

**Problem:** If all commis are dead (G18) and you cannot restart tmuxinator, the sprint is stuck.

**Fix in the Chef prompt**: an explicit Plan B:

```
IF THE COMMIS ARE DEAD:
1. Do NOT try to re-spawn (same error)
2. Do the work YOURSELF directly in this session
3. Use the existing worktrees (cd /path/to/worktree)
4. One job at a time (no parallelism) but keep moving
5. Tell the user: "Commis dead, continuing solo"
```

---

## G24 — No ccheck window = Chef blocks on every permission (CRITICAL)

**Problem:** The Chef receives permission prompts for every commis edit (shared-state.md, source files). Without a dedicated process watching and approving, the Chef sits idle waiting for a human Enter. In sprint grob-s2 this caused hour-long stalls.

**Cause:** The old design used a `/loop` in the user's terminal session. This was fragile (closing the terminal killed it), optional (Phase 5 said "launch" but didn't enforce it), and didn't survive terminal disconnects.

**Fix:** The ccheck (contre-chef) is now a **mandatory third tmux window** in the tmuxinator config. It starts and stops with the tmux session (`tmuxinator start/stop`), reads sensitive zones from shared-state.md on every tick, and auto-approves normal zones.

```yaml
# In tmuxinator YAML — MANDATORY window
- ccheck:
    root: {project_path}
    panes:
      - claude --dangerously-skip-permissions --permission-mode bypassPermissions --append-system-prompt "$(cat {project_path}/.claude/prompts/ccheck-{session_name}.md)"
```

**If the ccheck prompt is missing from the generated files → the skill output is INVALID.** Like the 3 voting Sous-Chefs, the ccheck is non-negotiable.

**If the user closes the ccheck window accidentally:** the Chef will block on the next permission. Relaunch with `tmuxinator stop && tmuxinator start` or manually open a new window and run the ccheck command.

---

## G25 — .claude/ trust guard survives --dangerously-skip-permissions

**Problem:** The `--dangerously-skip-permissions` flag bypasses tool permissions (Bash, Edit, Read, etc.). But the `.claude/` directory has a **separate trust guard** at a higher level. Bash commands that touch `.claude/` (e.g., `sed ... .claude/shared-state.md | awk`, `echo >> .claude/ccheck.log`) trigger a "Do you want to proceed?" prompt that is NOT covered by the bypass flags.

**Cause:** The `.claude/` directory contains settings, prompts, and shared state. Claude Code protects it with a workspace-level trust check independent of the permission system.

**Symptoms:** The ccheck blocks on commands like:
```
sed -n '...' .claude/shared-state.md | awk '{print $1}'
echo "$(date -Is) | APPROVE | ..." >> .claude/ccheck.log
```

**Fix in the ccheck prompt:**
1. Use **Read tool** (not `cat`/`sed`/`grep`) to read `.claude/` files
2. Use **Write tool** (not `echo >>`) to write to `.claude/ccheck.log`
3. Only use **Bash for tmux commands** (`tmux capture-pane`, `tmux send-keys`, `sleep`)
4. **One Bash command per call** — no `&&`, no `|`, no `;`

**Fix in settings.local.json** (belt AND suspenders):
```json
"Read(//{project_path}/.claude/**)",
"Write(//{project_path}/.claude/ccheck.log)",
"Write(//{project_path}/.claude/sprint-history/**)",
"Bash(awk:*)",
"Bash(date:*)"
```

Pre-authorizing these in settings.local.json covers the case where a commis (not just the ccheck) needs to read shared-state or write to sprint history.

**If the user closes the ccheck window accidentally:** the Chef will block on the next permission. Relaunch with `tmuxinator stop && tmuxinator start` or manually open a new window and run the ccheck command.
