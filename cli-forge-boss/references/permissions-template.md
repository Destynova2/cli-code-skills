# Template — Permissions

Generate at `{project}/.claude/settings.local.json`.

## Principle

The Chef and the commis must be able to work with ZERO permission prompts.
Every missing permission = the Chef blocks (G1).

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
      "Bash(sleep:*)",
      "Bash(tmux:*)",

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

      // === Shared state (absolute path — G5) ===
      "Edit(//{project_path}/.claude/shared-state.md)",

      // === External repos (G14) ===
      // "Read(//{external_repo_path}/**)",

      // === Side projects ===
      // "Write(//{side_project_path}/**)",
      // "Edit(//{side_project_path}/**)",
      // "Read(//{side_project_path}/**)",

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
- [ ] Absolute paths to external repos the commis must read
- [ ] Absolute paths to side projects the commis must write to
- [ ] CLI skills needed for the quality gates
