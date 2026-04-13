# Template — Permissions

Generate at `{project}/.claude/settings.local.json`.

## Principle

The Chef, Sous-Chef, Contre-Chef (ccheck), and commis must ALL work with ZERO permission prompts.
Every missing permission = an agent blocks (G1, G24).

**The `.claude/` directory has a special trust guard** that is NOT bypassed by `--dangerously-skip-permissions`. Commands reading/writing inside `.claude/` (shared-state, sprint-history, ccheck.log, prompts) trigger a separate trust prompt the first time. Pre-authorize these paths explicitly.

---

**Adapt to detected project type.** Only include build tools for the detected stack — do not leave commented-out tools for other stacks. Use the detection table from Phase 0.1.

| Project type | Include |
|-------------|---------|
| Rust | `Bash(cargo:*)` |
| Go | `Bash(go:*)`, `Bash(golangci-lint:*)` |
| JS/TS | `Bash(npm:*)`, `Bash(npx:*)`, `Bash(node:*)` |
| Python | `Bash(python3:*)`, `Bash(pip:*)`, `Bash(uv:*)`, `Bash(pytest:*)` |
| Terraform | `Bash(terraform:*)`, `Bash(tofu:*)` |
| Helm | `Bash(helm:*)`, `Bash(helmfile:*)` |
| Docker | `Bash(docker:*)`, `Bash(podman:*)` |
| Nix | `Bash(nix:*)` |
| Monorepo | union of detected sub-project types |

```json
{
  "permissions": {
    "allow": [
      // === Build & dev tools (generated from detected project type) ===
      // REPLACE with tools from the table above
      "Bash({build_tool}:*)",

      // === Git & CI ===
      "Bash(git:*)",
      "Bash(gh:*)",

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
      "Bash(tmux:*)",
      "Bash(ln:*)",

      // === File tools (cover every file in the project) ===
      "Read",
      "Edit",
      "Write",
      "Glob",
      "Grep",

      // === Web (if needed) ===
      "WebSearch",
      "WebFetch",

      // === CLI skills (quality gates) ===
      "Skill(cli-*)",

      // === .claude/ directory (G5 + G24 — trust guard bypass) ===
      // These are MANDATORY. Without them the Chef and ccheck block
      // on every shared-state edit, sprint-history write, and log append.
      "Edit(//{project_path}/.claude/shared-state.md)",
      "Read(//{project_path}/.claude/**)",
      // ccheck.log now lives in /tmp/ to avoid .claude/ trust guard (G25)
      "Write(//{project_path}/.claude/sprint-history/**)",
      "Bash(mkdir -p //{project_path}/.claude/sprint-history/*)",
      "Bash(ln -sfn * //{project_path}/.claude/sprint-history/current)",

      // === External repos (G14) ===
      // "Read(//{external_repo_path}/**)",

      // === Side projects ===
      // "Write(//{side_project_path}/**)",
      // "Edit(//{side_project_path}/**)",
      // "Read(//{side_project_path}/**)",

      // === Obsidian vault (G22) ===
      // "Edit(//{obsidian_vault_path}/**)",
      // "Read(//{obsidian_vault_path}/**)",

      // === Team tools ===
      "TeamCreate",
      "TeamDelete",
      "SendMessage",
      "TaskCreate",
      "TaskUpdate",
      "TaskList",
      "TaskGet"
    ],
    "deny": [
      "Bash(rm -rf /:*)",
      "Bash(rm -rf /*:*)"
    ]
  }
}
```

## Checklist

Before generating, verify:

- [ ] All the project's build tools (cargo, npm, python, make, etc.)
- [ ] Absolute path to shared-state.md
- [ ] `.claude/` directory permissions (Read, Write, Edit — G5 + G24)
- [ ] Sprint history paths (mkdir, ln, Write)
- [ ] ccheck.log Write permission
- [ ] `awk` and `date` in shell basics (used by ccheck for parsing + timestamps)
- [ ] Absolute paths to external repos the commis must read
- [ ] Absolute paths to side projects the commis must write to
- [ ] Obsidian vault paths if applicable (G22)
- [ ] CLI skills needed for the quality gates
