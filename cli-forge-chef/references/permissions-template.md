# Template — Permissions (multi-role, per-worktree)

Generate ONE `settings.local.json` per role, each inside the role's own worktree.
Claude Code reads `.claude/settings.local.json` from the agent's CWD, so placing
each agent in its own worktree naturally gives it its own permission set.

## Principle

**One role = one worktree = one `settings.local.json`.**

Previously, a single file at the repo root granted every tool to every agent.
A hallucinated commis could then run `tofu apply` or `git push --force origin main`
even though the prompt told it not to. The fix is filesystem-level: each commis
cannot *physically* execute the dangerous command because its own
`settings.local.json` denies the Bash prefix.

The quorum (3 voting sous-chefs) still votes on diffs AND on sensitive shell
commands (see `apply-quorum.md`), but the per-worktree permissions are the
safety net if the protocol is bypassed.

**The `.claude/` directory has a special trust guard** that is NOT bypassed by
`--dangerously-skip-permissions`. Commands reading/writing inside `.claude/`
(shared-state, sprint-history, ccheck.log, prompts) trigger a separate trust
prompt the first time. Pre-authorize these paths explicitly in every role file
that needs them.

---

## Worktree layout

The Chef creates these worktrees in Phase 0 (tmuxinator `on_project_start` or
manually before launch). Each path below corresponds to one agent's CWD.

```
{project}/                          ← Chef CWD (repo root, read-only baseline)
{project}-wt-gate/                  ← Sous-Chef Merge
{project}-wt-maitre/                ← Maître d'hôtel
{project}-wt-apply/                 ← Apply pane (tofu/helm/kubectl)
{project}-wt-vote-scope/            ← sous-chef-scope (read-only)
{project}-wt-vote-secu/             ← sous-chef-secu (read-only)
{project}-wt-vote-qualite/          ← sous-chef-qualite (read-only)
{project}-wt-commis-1/              ← commis 1
{project}-wt-commis-2/              ← commis 2
...
```

The Chef's `TeamCreate` / Agent spawn MUST pass `cwd: "<worktree path>"` so the
agent picks up the correct settings file (see `chef-prompt-template.md` §3).

---

## Files to generate (one per role)

Adapt to detected project type. Only include build tools for the detected stack.

| Project type | Commis `allow` additions |
|-------------|---------|
| Rust | `Bash(cargo:*)` |
| Go | `Bash(go:*)`, `Bash(golangci-lint:*)` |
| JS/TS | `Bash(npm:*)`, `Bash(npx:*)`, `Bash(node:*)` |
| Python | `Bash(python3:*)`, `Bash(pip:*)`, `Bash(uv:*)`, `Bash(pytest:*)` |
| Terraform | `Bash(tofu plan:*)`, `Bash(tofu validate:*)`, `Bash(terraform plan:*)`, `Bash(terraform validate:*)` — **plan/validate only, never apply** |
| Helm | `Bash(helm template:*)`, `Bash(helm lint:*)`, `Bash(helm diff:*)` — **template/lint/diff only, never upgrade** |
| Kubernetes | `Bash(kubectl get:*)`, `Bash(kubectl describe:*)`, `Bash(kubectl diff:*)` — **read + diff only, never apply** |
| Docker | `Bash(docker build:*)`, `Bash(podman build:*)` — **build only** |

Key principle: the **commis** gets the read-only / validate-only / plan-only
side of every toolchain. The **apply pane** gets the mutating side, and only
after a quorum vote (see §Apply pane below).

---

## File 1 — Commis (`{project}-wt-commis-N/.claude/settings.local.json`)

One file per commis. Replace `{N}` and `{commis_worktree_path}` when generating.

```json
{
  "permissions": {
    "allow": [
      // === Build & dev tools (project-type dependent) ===
      "Bash({build_tool}:*)",

      // === Local git only (NO push, NO force, NO rebase on base) ===
      "Bash(git status:*)",
      "Bash(git diff:*)",
      "Bash(git log:*)",
      "Bash(git add:*)",
      "Bash(git commit:*)",
      "Bash(git branch:*)",
      "Bash(git checkout:*)",
      "Bash(git reset --soft:*)",
      "Bash(git reset --mixed:*)",
      "Bash(git stash:*)",
      "Bash(git show:*)",
      "Bash(git worktree list:*)",

      // === Read-only GitHub (no PR merge, no release) ===
      "Bash(gh pr view:*)",
      "Bash(gh pr list:*)",
      "Bash(gh issue view:*)",
      "Bash(gh issue list:*)",
      "Bash(gh run view:*)",
      "Bash(gh run list:*)",

      // === Shell basics ===
      "Bash(ls:*)",
      "Bash(mkdir:*)",
      "Bash(cp:*)",
      "Bash(cat:*)",
      "Bash(echo:*)",
      "Bash(head:*)",
      "Bash(tail:*)",
      "Bash(grep:*)",
      "Bash(find:*)",
      "Bash(sort:*)",
      "Bash(wc:*)",
      "Bash(sed:*)",
      "Bash(awk:*)",
      "Bash(date:*)",
      "Bash(sleep:*)",

      // === File tools ===
      "Read",
      "Edit",
      "Write",
      "Glob",
      "Grep",

      // === Team protocol (to request quorum votes) ===
      "SendMessage",

      // === .claude/ directory (trust guard) ===
      "Edit(//{project_path}/.claude/shared-state.md)",
      "Read(//{project_path}/.claude/**)",
      "Read(//{commis_worktree_path}/.claude/**)",

      // === Web (if allowed) ===
      "WebSearch",
      "WebFetch",

      // === Quality gate skills the commis may self-run ===
      "Skill(cli-audit-code)",
      "Skill(cli-audit-test)",
      "Skill(cli-git-conventional)"
    ],
    "deny": [
      // === Git write operations — Sous-Chef's job ===
      "Bash(git push:*)",
      "Bash(git push --force:*)",
      "Bash(git push --force-with-lease:*)",
      "Bash(git rebase:*)",
      "Bash(git merge:*)",
      "Bash(git fetch:*)",
      "Bash(git pull:*)",
      "Bash(git reset --hard:*)",

      // === PR / release operations — Sous-Chef's / Maître's job ===
      "Bash(gh pr merge:*)",
      "Bash(gh pr create:*)",
      "Bash(gh pr close:*)",
      "Bash(gh release:*)",
      "Bash(gh api -X DELETE:*)",
      "Bash(gh api --method DELETE:*)",
      "Bash(gh api -X PATCH:*)",
      "Bash(gh api --method PATCH:*)",

      // === Infra apply — apply pane's job, after quorum ===
      "Bash(tofu apply:*)",
      "Bash(tofu destroy:*)",
      "Bash(terraform apply:*)",
      "Bash(terraform destroy:*)",
      "Bash(helm upgrade:*)",
      "Bash(helm install:*)",
      "Bash(helm uninstall:*)",
      "Bash(helm rollback:*)",
      "Bash(kubectl apply:*)",
      "Bash(kubectl delete:*)",
      "Bash(kubectl replace:*)",
      "Bash(kubectl patch:*)",
      "Bash(kubectl rollout:*)",
      "Bash(ansible-playbook:*)",

      // === Destructive defaults ===
      "Bash(rm -rf /:*)",
      "Bash(rm -rf /*:*)",
      "Bash(rm -rf ~:*)"
    ]
  }
}
```

---

## File 2 — Sous-Chef Merge (`{project}-wt-gate/.claude/settings.local.json`)

Gets git push + PR operations. Does NOT get infra apply or force-push.

```json
{
  "permissions": {
    "allow": [
      "Bash({build_tool}:*)",

      // === Full git EXCEPT force-push to any branch ===
      "Bash(git:*)",

      // === Full gh EXCEPT release + api DELETE/PATCH ===
      "Bash(gh pr:*)",
      "Bash(gh run:*)",
      "Bash(gh issue:*)",
      "Bash(gh repo view:*)",
      "Bash(gh api repos/*/pulls:*)",
      "Bash(gh api repos/*/branches:*)",
      "Bash(gh api repos/*/rulesets:*)",

      // === Shell basics ===
      "Bash(ls:*)", "Bash(mkdir:*)", "Bash(cp:*)", "Bash(cat:*)",
      "Bash(echo:*)", "Bash(head:*)", "Bash(tail:*)", "Bash(grep:*)",
      "Bash(find:*)", "Bash(sort:*)", "Bash(wc:*)", "Bash(sed:*)",
      "Bash(awk:*)", "Bash(date:*)", "Bash(sleep:*)", "Bash(tmux:*)",

      "Read", "Edit", "Write", "Glob", "Grep",
      "SendMessage", "TaskCreate", "TaskUpdate", "TaskList", "TaskGet",

      "Edit(//{project_path}/.claude/shared-state.md)",
      "Read(//{project_path}/.claude/**)",
      "Write(//{project_path}/.claude/sprint-history/**)",
      "Bash(mkdir -p //{project_path}/.claude/sprint-history/*)",
      "Bash(ln -sfn * //{project_path}/.claude/sprint-history/current)",

      "Skill(cli-audit-*)",
      "Skill(cli-git-conventional)",
      "Skill(cli-forge-github)"
    ],
    "deny": [
      // === Force-push is maître d'hôtel's job, feature branches only ===
      "Bash(git push --force:*)",
      "Bash(git push --force-with-lease:*)",
      "Bash(git push -f:*)",

      // === Infra apply is apply pane's job ===
      "Bash(tofu apply:*)",
      "Bash(tofu destroy:*)",
      "Bash(terraform apply:*)",
      "Bash(terraform destroy:*)",
      "Bash(helm upgrade:*)",
      "Bash(helm install:*)",
      "Bash(helm uninstall:*)",
      "Bash(kubectl apply:*)",
      "Bash(kubectl delete:*)",
      "Bash(ansible-playbook:*)",

      // === Destructive defaults ===
      "Bash(rm -rf /:*)",
      "Bash(rm -rf /*:*)"
    ]
  }
}
```

---

## File 3 — Maître d'hôtel (`{project}-wt-maitre/.claude/settings.local.json`)

Rebase + force-with-lease on **feature branches only**. Never on base branches.

```json
{
  "permissions": {
    "allow": [
      "Bash(git fetch:*)",
      "Bash(git rebase:*)",
      "Bash(git push:*)",
      "Bash(git push --force-with-lease:*)",
      "Bash(git checkout:*)",
      "Bash(git status:*)",
      "Bash(git log:*)",
      "Bash(git diff:*)",
      "Bash(git branch:*)",
      "Bash(git worktree list:*)",

      "Bash(gh pr view:*)",
      "Bash(gh pr merge:*)",
      "Bash(gh pr list:*)",
      "Bash(gh run view:*)",
      "Bash(gh run list:*)",
      "Bash(gh run rerun:*)",
      "Bash(gh release list:*)",
      "Bash(gh release view:*)",
      "Bash(gh api repos/*/git/refs/heads/*:*)",

      "Bash(ls:*)", "Bash(cat:*)", "Bash(grep:*)", "Bash(awk:*)",
      "Bash(sed:*)", "Bash(date:*)", "Bash(sleep:*)",

      "Read", "Edit",
      "SendMessage",

      "Edit(//{project_path}/.claude/shared-state.md)",
      "Read(//{project_path}/.claude/**)"
    ],
    "deny": [
      // === NEVER force-push any base branch ===
      // Base branches are named main, master, develop, trunk, release/*
      // The matcher below is conservative — the Maître's prompt has the
      // authoritative list. This is the filesystem fallback.
      "Bash(git push --force origin main:*)",
      "Bash(git push --force origin master:*)",
      "Bash(git push --force origin develop:*)",
      "Bash(git push --force origin trunk:*)",
      "Bash(git push --force-with-lease origin main:*)",
      "Bash(git push --force-with-lease origin master:*)",
      "Bash(git push --force-with-lease origin develop:*)",
      "Bash(git push --force-with-lease origin trunk:*)",
      "Bash(git push -f origin main:*)",
      "Bash(git push -f origin master:*)",
      "Bash(git push -f origin develop:*)",

      // === No code edits, no apply ===
      "Bash(tofu:*)", "Bash(terraform:*)",
      "Bash(helm upgrade:*)", "Bash(helm install:*)",
      "Bash(kubectl apply:*)", "Bash(kubectl delete:*)",
      "Write",
      "Bash(rm -rf /:*)",
      "Bash(rm -rf /*:*)"
    ]
  }
}
```

---

## File 4 — Apply pane (`{project}-wt-apply/.claude/settings.local.json`)

The only role allowed to run `tofu apply` / `helm upgrade` / `kubectl apply`.
Every apply MUST be preceded by a quorum vote on the plan output (see
`apply-quorum.md`). The protocol is enforced in the prompt, the permissions are
the safety net.

```json
{
  "permissions": {
    "allow": [
      // === Infra tooling (plan AND apply) ===
      "Bash(tofu:*)",
      "Bash(terraform:*)",
      "Bash(helm:*)",
      "Bash(helmfile:*)",
      "Bash(kubectl:*)",
      "Bash(kustomize:*)",
      "Bash(ansible:*)",
      "Bash(ansible-playbook:*)",

      // === Read-only git (to checkout the right ref) ===
      "Bash(git status:*)",
      "Bash(git log:*)",
      "Bash(git checkout:*)",
      "Bash(git pull --ff-only:*)",
      "Bash(git fetch:*)",

      // === Shell basics ===
      "Bash(ls:*)", "Bash(cat:*)", "Bash(echo:*)", "Bash(grep:*)",
      "Bash(find:*)", "Bash(date:*)", "Bash(sleep:*)",

      "Read", "Edit",
      "SendMessage",

      "Edit(//{project_path}/.claude/shared-state.md)",
      "Read(//{project_path}/.claude/**)"
    ],
    "deny": [
      // === NO code commit, NO PR ops, NO release ===
      "Bash(git push:*)",
      "Bash(git commit:*)",
      "Bash(git rebase:*)",
      "Bash(gh pr:*)",
      "Bash(gh release:*)",

      "Bash(rm -rf /:*)",
      "Bash(rm -rf /*:*)"
    ]
  }
}
```

---

## File 5 — 3 voting sous-chefs (`{project}-wt-vote-{scope|secu|qualite}/.claude/settings.local.json`)

Read-only. They vote via SendMessage, never execute.

```json
{
  "permissions": {
    "allow": [
      "Read", "Glob", "Grep",
      "SendMessage",

      "Bash(ls:*)",
      "Bash(cat:*)",
      "Bash(head:*)",
      "Bash(tail:*)",
      "Bash(grep:*)",
      "Bash(find:*)",
      "Bash(git diff:*)",
      "Bash(git log:*)",
      "Bash(git show:*)",
      "Bash(git status:*)",

      "Read(//{project_path}/.claude/**)",

      "Skill(cli-audit-code)",
      "Skill(cli-audit-drift)",
      "Skill(cli-audit-test)"
    ],
    "deny": [
      "Write", "Edit",
      "Bash(git add:*)",
      "Bash(git commit:*)",
      "Bash(git push:*)",
      "Bash(git rebase:*)",
      "Bash(git merge:*)",
      "Bash(git reset:*)",
      "Bash(gh:*)",
      "Bash(tofu:*)", "Bash(terraform:*)",
      "Bash(helm:*)", "Bash(kubectl:*)", "Bash(ansible:*)",
      "Bash(rm:*)"
    ]
  }
}
```

---

## File 6 — Chef root (`{project}/.claude/settings.local.json`)

The Chef plans and decides. It should never execute mutating shell commands
directly — delegation is its whole point. This file is the most restrictive.

```json
{
  "permissions": {
    "allow": [
      "Read", "Glob", "Grep",
      "SendMessage", "TeamCreate", "TeamDelete",
      "TaskCreate", "TaskUpdate", "TaskList", "TaskGet",

      "Bash(ls:*)", "Bash(cat:*)", "Bash(grep:*)", "Bash(find:*)",
      "Bash(git log:*)", "Bash(git status:*)", "Bash(git diff:*)",
      "Bash(git branch:*)", "Bash(git worktree list:*)",
      "Bash(gh repo view:*)", "Bash(gh issue list:*)", "Bash(gh pr list:*)",
      "Bash(tmux send-keys:*)",
      "Bash(date:*)", "Bash(sleep:*)",

      "Edit(//{project_path}/.claude/shared-state.md)",
      "Read(//{project_path}/.claude/**)",
      "Write(//{project_path}/.claude/sprint-history/**)",
      "Bash(mkdir -p //{project_path}/.claude/sprint-history/*)",
      "Bash(ln -sfn * //{project_path}/.claude/sprint-history/current)",

      "Skill(cli-*)"
    ],
    "deny": [
      // === The Chef never writes code, never pushes, never applies ===
      "Bash(git add:*)",
      "Bash(git commit:*)",
      "Bash(git push:*)",
      "Bash(git merge:*)",
      "Bash(git rebase:*)",
      "Bash(git reset:*)",
      "Bash(gh pr create:*)",
      "Bash(gh pr merge:*)",
      "Bash(gh release:*)",
      "Bash(tofu:*)", "Bash(terraform:*)",
      "Bash(helm upgrade:*)", "Bash(helm install:*)",
      "Bash(kubectl apply:*)", "Bash(kubectl delete:*)",
      "Bash(ansible-playbook:*)",
      "Bash(rm -rf /:*)",
      "Bash(rm -rf /*:*)"
    ]
  }
}
```

---

## Generator checklist

Before launching the brigade, verify:

- [ ] Every worktree exists (`git worktree list` shows all of them)
- [ ] Every worktree has `.claude/settings.local.json`
- [ ] Every Agent spawn in the Chef prompt passes the right `cwd` (see `chef-prompt-template.md`)
- [ ] The commis file denies `Bash(git push:*)`, `Bash(tofu:*)`, `Bash(helm upgrade:*)`, `Bash(kubectl apply:*)`
- [ ] The gate file denies `Bash(tofu:*)`, `Bash(helm upgrade:*)`, `Bash(kubectl apply:*)`, `Bash(git push --force:*)`
- [ ] The maître file denies force-push on `origin main|master|develop|trunk`
- [ ] The apply file denies `Bash(git push:*)`, `Bash(git commit:*)`
- [ ] The 3 voting sous-chef files deny `Write`, `Edit`, and all mutating Bash
- [ ] The Chef root file denies `Bash(git push:*)`, `Bash(tofu:*)`, `Bash(helm upgrade:*)`
- [ ] External repos the commis read are pre-authorized per-commis (G14)
- [ ] Obsidian vault paths are added where needed (G22)

## Why one file per worktree beats one global file

| Scenario | Old (one global file) | New (per-worktree) |
|---|---|---|
| Commis hallucinates `tofu apply -auto-approve` | Runs — destroys infra | Denied by filesystem, prompt ignored |
| Sous-Chef Merge runs `tofu apply` by mistake | Runs | Denied |
| Apply pane runs `git push origin main --force` | Runs | Denied |
| Voting sous-chef is prompt-injected to `git commit -am "bypass"` | Runs | Denied |
| Prompt says "do not push" but agent does anyway | Pushes | Denied |

The prompts still tell each role what it *should* do. The permission files
stop each role from doing what it *should not*, even if something goes wrong
at the prompt layer.
