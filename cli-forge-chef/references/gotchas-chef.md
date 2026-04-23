# Gotchas — Multi-Agent Orchestration

> **Rule:** The Chef prompt MUST include this section verbatim. Every gotcha comes from a real failure.
>
> **Terminology:** "Chef" = the planning agent (Brigade role). "conductor" = the tmux window/pane name. They are the same agent. Never use "boss" in prompts or configs — it's only the skill name (`cli-forge-chef`).

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

## G2 — Use `--append-system-prompt-file`, NOT `--append-system-prompt "$(cat ...)"`

**Problem:** Loading a prompt from a file via `--append-system-prompt "$(cat prompt.md)"` re-parses the markdown through the shell. A realistic Chef prompt contains 100+ backticks, many `$(...)` snippets, and mixed quotes — the shell mangles it, claude fails to launch (see G34 for details).

**Fix:** The CLI natively exposes file-loading flags:

```bash
claude --append-system-prompt-file path/to/prompt.md    # append to default
claude --system-prompt-file path/to/prompt.md            # replace default
```

The flag reads the file INSIDE claude — the shell sees only the path, never the content. This is the canonical way to load a long prompt from disk.

**Historical note:** earlier versions of this gotcha claimed `--system-prompt-file` did not exist. That was wrong, corroborated by the official CLI reference (https://code.claude.com/docs/en/cli). Use the native flag.

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

---

## G26 — Protected branch rejects direct push (CRITICAL)

**Problem:** develop (or main) has a GitHub ruleset requiring "Changes must be made through a pull request". The Sous-Chef tries to `git push origin develop` after merging locally → GitHub rejects the push. All commis work is stranded locally, nothing reaches the remote.

**Cause:** The merge protocol assumed direct push was allowed. It didn't check whether the base branch was protected.

**Symptoms:**
```
remote: error: GH006: Protected branch update failed
remote: error: Changes must be made through a pull request.
```

**Fix in the Sous-Chef merge protocol:**
1. At startup, detect if the base branch requires PRs:
   ```bash
   gh api repos/{owner}/{repo}/rulesets 2>/dev/null | grep -q 'pull_request'
   ```
2. If YES → **PR mode**: push the commis branch, `gh pr create`, `gh pr checks --watch`, `gh pr merge --squash`
3. If NO → **direct mode**: merge locally, push directly

**Detection:** If you see `GH006: Protected branch update failed` in the gate window, switch to PR mode immediately. Do NOT try to force-push.

**Impact on the release chain:** Once PRs are merged to develop, the chain is automatic:
release-plz PR → tag → sync-main PR → main catches up (10-30 min).
The Sous-Chef should NOT wait for the full chain — once the PR to develop is merged, the commis can continue.

## G30 — Commis running `git fetch origin` or `git reset --hard` corrupts the brigade

**Problem:** A commis in its worktree runs `git fetch origin` to "stay up to date", or `git reset --hard HEAD~1` to "undo a botched commit". The Sous-Chef then tries to push or rebase the branch and either (a) hits a moving target (fetch introduced remote refs the Sous-Chef did not plan for), or (b) discovers the branch is missing commits it expected (reset --hard destroyed them).

**Cause:** Commis prompts did not forbid these operations. Commis assume that their worktree is "theirs" to manage, when in fact the Sous-Chef owns the sync and rebase protocol — the commis only owns the working tree and commits.

**Symptoms:**
- Sous-Chef reports "branch is behind / ahead of origin in an unexpected way"
- `git log` on the commis branch shows missing commits the commis remembers writing
- Push fails with "tip of your current branch is behind its remote counterpart" because a stale fetch moved origin refs
- Rebase by the Sous-Chef produces spurious conflicts on files the commis never touched

**Fix in the commis prompt (chef-prompt-template.md §Commis prompt template):** Add a FORBIDDEN git operations block:
- NEVER `git fetch origin` or `git pull`
- NEVER `git reset --hard`
- If reset is truly needed: `git reset --soft` or `git reset --mixed` only, then notify the Sous-Chef

**Detection:** At the start of a sprint, the Sous-Chef can verify by checking `git reflog` on each commis branch after the first commit — any `reset: moving to HEAD~` or `fetch origin` entry is a red flag. If detected, SendMessage the commis to stop and escalate to the Chef.

**Why this matters:** The brigade is deterministic by design — one commis = one branch = one PR. Any git operation that silently rewrites history or moves remote refs breaks that determinism and turns a 2h sprint into a 6h forensic session.

## G31 — Chef pane idle because no initial user message (CRITICAL)

**Symptom:** The Chef pane shows the `❯` prompt but nothing happens. The `chef-prompt.md` system prompt contains "G3: Start IMMEDIATELY without waiting for a user message" yet the Chef sits there. Sprint never starts.

**Root cause:** `claude --append-system-prompt "..."` without a trailing **positional prompt argument** launches the REPL in idle state. The system prompt only governs behaviour *when the user speaks* — Claude Code's interactive mode does not auto-fire. `tmux send-keys "text" Enter` workarounds are unreliable: the input widget treats the keystroke as a newline within the multiline buffer rather than a submission, leaving the message typed-but-unsent.

**Fix:** Pass the kick-off as the last positional argument:

```yaml
- claude --dangerously-skip-permissions --permission-mode bypassPermissions \
         --teammate-mode tmux \
         --append-system-prompt "$(cat prompts/chef-sprint.md)" \
         "Demarre IMMEDIATEMENT Phase 0 selon le system prompt : TeamCreate, recruter sous-chefs + maitre-d + commis, puis executer le PERT."
```

The last string is consumed as the first **user message** — Claude fires the system prompt in response and the sprint boots. Same trick applies to `ccheck` and `contre-chef-inter` panes.

**Detection:** 10 min after `tmuxinator start`, `gh pr list --state open --limit 10` is still empty AND `tmux capture-pane -t <sprint>:chef -p | tail -5` shows no output from Claude. If so, rerun the pane command with the positional prompt appended.

## G32 — Agent Teams teammates inherit the lead's permissions (cannot be isolated per-worktree)

**Problem:** A natural expectation from the « one worktree per role » pattern is that each teammate reads its own `.claude/settings.local.json` — so a commis would be filesystem-level denied from running `git push` or `tofu apply`. In reality, **Agent Teams teammates do NOT accept `cwd:` or `isolation:` on spawn**, and per the official doc: « all teammates start with the lead's permission mode ». The Chef's root `settings.local.json` is the ONLY one that applies to every teammate.

**Consequence:** adding `cwd: "<worktree>"` to an Agent spawn is silently ignored. A commis prompted to work in `{project}-wt-commis-scaleway/` still inherits the Chef's root permissions, not the commis-scaleway permission file. If the skill's generator claims per-teammate isolation, the claim is false.

**What actually works for isolation:**
- Standalone `claude` sessions launched as tmuxinator panes (chef, ccheck, contre-chef-inter, apply pane) DO read their own cwd's `settings.local.json`. They are separate processes, not teammates.
- The `apply` pane in particular must be a standalone pane (not a teammate) so its permissions (`tofu:*`, `helm:*`, `kubectl:*`) are truly isolated from the commis world.

**What does NOT work:**
- `cwd:` on `Agent { ... }` spawns — ignored, not a supported parameter in Agent Teams.
- Per-worktree `settings.local.json` for commis / sous-chefs spawned via TeamCreate — ignored, they use the lead's file.
- Expecting a commis to be blocked at the Bash level from `tofu apply` — it won't, unless the Chef's root permission file denies it too.

**Recommended practice:**
1. Put the strictest baseline in the Chef's root `settings.local.json`. Deny `git push`, `tofu apply`, `helm upgrade`, `kubectl apply`, `gh release` at the root — every teammate inherits the deny.
2. Allow those commands only in the standalone panes that need them: `gate` (push, pr merge), `apply` (tofu/helm/kubectl), `maitre` (force-with-lease on feature branches).
3. Tell each commis in its prompt to `cd {worktree_path}` as its first action so Bash runs inside its worktree even though permissions come from the Chef.

**Detection:** look for `cwd:` in generated chef prompt. If present, it's cosmetic — the agent will still inherit the Chef's permissions. Grep: `grep -c 'cwd:' .claude/prompts/chef-*.md` should be 0.

**Alternative if true per-role isolation is needed:** abandon Agent Teams, launch each role as its own standalone `claude` session in its own tmuxinator pane, coordinate via `shared-state.md` + `.claude/votes/` polling. This is mentioned in the skill's readme as the « file-coord mode » future option — not implemented yet.

## G33 — Apply commands need their own standalone pane (NOT a teammate)

**Problem:** `tofu apply`, `helm upgrade`, `kubectl apply`, `gh release create` are irreversible and must be gated by a quorum vote. If the apply executor is an Agent Teams teammate, it inherits the Chef's permissions — which the skill denies for everyone (per G32). So the teammate cannot run apply even after the vote approves it.

**Fix:** the `apply` pane is a separate tmuxinator pane with its own `{project}-wt-apply/.claude/settings.local.json` that allows `tofu:*`, `helm:*`, `kubectl:*`, `ansible:*`. It's a standalone `claude` (or a pure-shell pane driven via `tmux send-keys`), NOT spawned via `TeamCreate`. This is the only place the apply primitives are allowed on the whole machine for the brigade.

**The quorum protocol (see `references/apply-quorum.md`) works like this:**
1. A commis wants to apply — it SendMessages the apply pane with the command + justification.
2. The apply pane runs `tofu plan` / `helm diff` / `kubectl diff` and captures the output.
3. The apply pane SendMessages the 3 voting sous-chefs with the plan for vote.
4. On 3/3 APPROVE, the apply pane executes the command (its settings.local.json allows it).
5. On any DENY, the apply pane forwards DENY + reason back to the commis.

**Detection:** `test -f {project}-wt-apply/.claude/settings.local.json && grep -q 'tofu:\*\|helm:\*\|kubectl:\*' $_`. And verify no commis settings file allows those.

## G34 — Shell-mangling of the prompt via `$(cat ...)` (SOLVED by --append-system-prompt-file)

**Historical symptom:** `tmuxinator start <session>` appears to succeed, but the claude pane is dead or in a strange state — you see bash error messages, unmatched quotes, or the pane showing the raw text of the prompt mid-command line. A realistic Chef prompt with 96 backticks + 106 quotes in one `.md` file would trigger this.

**Root cause:** `claude --append-system-prompt "$(cat prompts/chef.md)"` expands `$(cat ...)` through the shell first, then re-parses the content. The shell reinterprets backticks / `$(...)` / unbalanced quotes inside the markdown as its own metacharacters.

**Fix (current):** Use `--append-system-prompt-file <path>` instead — claude reads the file internally, the shell only sees the path:

```yaml
- claude --append-system-prompt-file .claude/prompts/chef-session.md "Kick-off message."
```

No wrapper script, no `$(< file)` workaround, no `.initial.txt` — all of that was
overkill once we discovered the native flag. See G2 for the full CLI reference.

**Previous fix (obsolete):** the skill used to generate `scripts/brigade-launch-agent.sh` that wrapped `claude` with `$(< file)`. That script is no longer emitted — if you have an old copy, delete it.

**Detection:** `! grep -q '\$(cat ' ~/.config/tmuxinator/<session>.yml` — if `$(cat ...)` appears anywhere, the generator is outdated.

## G35 — `git worktree add ... | head -1` under `pipefail` kills git by SIGPIPE

**Symptom:** The tmuxinator `on_project_start` runs, reports success, but half the worktrees are missing. Subsequent runs keep failing because orphan directories prevent re-creation. You chase ghosts for an hour.

**Root cause:** `set -o pipefail` propagates the rightmost non-zero exit status. When `head -1` has printed one line it closes stdin — git receives SIGPIPE on its next write and aborts. The worktree creation is interrupted mid-flight: the directory gets `rm -rf`'d but git's internal registry may be half-written. Even if `|| true` swallows the error, the damage is done.

**Fix:** Never pipe `git worktree add` into `head`. Use explicit redirection + explicit success/failure echo:

```bash
git worktree add "${wt_path}" -b "${branch}" >/dev/null 2>&1 \
  && echo "  + ${wt_path} → ${branch}" \
  || echo "  ! FAILED ${wt_path}" >&2
```

And **do not set `pipefail`** in the setup script — `set -eu` is enough, and `set -eu` alone does not turn SIGPIPE into a script-killing error.

The skill's `scripts/brigade-setup-worktrees.sh` follows this pattern from the start.

**Detection:** `git worktree list | wc -l` after `tmuxinator start` — should match the expected count (6 role worktrees + N commis + 1 main). Any discrepancy = G35.

**Related recovery:** the setup script detects orphan dirs (dir exists but git doesn't know) and `rm -rf`s them before re-adding. One re-run of the script is enough to recover from a G35 incident.

## G36 — Multiple commis sharing one branch = silent `git worktree add` failures

**Symptom:** One commis starts working, the others sit idle with "I cannot find my worktree" or their panes show the Chef's root dir instead of the expected commis worktree. Only the first commis gets a functional setup.

**Root cause:** `git worktree add <path> <branch>` fails with `fatal: '<branch>' is already checked out at '<other-path>'` if the branch is already associated with another worktree. When the generator assigns the SAME branch (e.g., `chore/phase-a-<session>`) to all N commis, only the first `git worktree add` succeeds — the remaining N-1 fail silently under `|| true`.

**Fix:** One branch per commis. The generator produces:

```
chore/phase-a-<session>            ← integration branch (no worktree, Sous-Chef Merge target)
chore/phase-a-<session>-bootstrap  ← commis-bootstrap's branch
chore/phase-a-<session>-scaleway   ← commis-scaleway's branch
chore/phase-a-<session>-k8s-day1   ← commis-k8s-day1's branch
...
```

Each commis works on its own branch in its own worktree. At "ready for merge", the Sous-Chef Merge consolidates the per-commis branches onto the integration branch (one merge per commis, sequential).

**Detection:** `git branch | grep chore/phase-a-<session> | wc -l` should equal `(number of commis) + 1`. If it equals 1, the generator is still using the old shared-branch pattern.

## G37 — First-time trust prompt appears even with `bypassPermissions`

**Symptom:** The chef pane shows `╭─ Bypass Permissions │ I trust the files ... │ No, exit ─╮` instead of starting work. `--dangerously-skip-permissions --permission-mode bypassPermissions` was supposed to skip this.

**Root cause:** Claude Code shows a first-time trust dialog for any working directory it hasn't seen before. The CLI flags bypass *permissions* (tool allow/deny) but not the initial *trust* acknowledgment. This is a one-time-per-cwd prompt — once accepted, subsequent launches skip it.

**Fix:** Have the tmuxinator `on_project_start` hook send `Down Enter` to each claude pane 3 s after launch:

```yaml
on_project_start:
  - (sleep 3 && for w in chef ccheck inter; do tmux send-keys -t {session}:$w Down Enter; done) &
```

The `&` is critical — without it tmuxinator blocks waiting for the sleep. The Chef pane must exist before the send-keys fires, hence the 3 s delay.

**Detection:** 30 s after launch, `tmux capture-pane -p -t <session>:chef | grep -c 'Bypass Permissions'` — if > 0, the trust prompt is still up and the Chef is not working.

## G38 — Empty bash array + `set -u` on bash 3.2 (OBSOLETE)

**Status:** OBSOLETE since the skill no longer generates `brigade-launch-agent.sh`. The bug was in that wrapper's array handling on macOS bash 3.2. With `--append-system-prompt-file` (see G34), no wrapper is generated, so the bug cannot happen.

**Kept here for archival reference:** if you write any bash script that uses arrays + `set -u`, always use `${arr[@]+"${arr[@]}"}` (conditional expansion) instead of `"${arr[@]}"`. The conditional form expands to nothing when the array is empty, instead of raising "unbound variable" on bash 3.2 (the macOS default).

**Why this matters:** Without the fix, every brigade start requires a human to attach to tmux and type "go". That defeats the unattended-sprint value proposition and burns 30-60 min of wall-clock per sprint waiting to be noticed.
