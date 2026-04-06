# Template — Permissions

Generer dans `{project}/.claude/settings.local.json`.

## Principe

Le boss et les workers doivent pouvoir travailler SANS AUCUN prompt de permission.
Chaque permission manquante = le boss se bloque (G1).

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

      // === File tools (couvrent tous les fichiers du projet) ===
      "Read",
      "Edit",
      "Write",
      "Glob",
      "Grep",

      // === Web (si necessaire) ===
      "WebSearch",
      "WebFetch",

      // === Skills CLI (quality gates) ===
      "Skill(cli-*)",

      // === Shared state (chemin absolu — G5) ===
      "Edit(//{project_path}/.claude/shared-state.md)",

      // === Repos externes (G14) ===
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

Avant de generer, verifier :

- [ ] Tous les outils build du projet (cargo, npm, python, make, etc.)
- [ ] Chemin absolu vers shared-state.md
- [ ] Chemins absolus vers les repos externes que les workers doivent lire
- [ ] Chemins absolus vers les side projects que les workers doivent ecrire
- [ ] Skills CLI necessaires pour les quality gates
