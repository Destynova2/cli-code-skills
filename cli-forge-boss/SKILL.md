---
name: cli-forge-boss
description: >
  Generate a multi-agent orchestrator for any project using the Brigade de Cuisine pattern:
  Chef (plans, decides), Sous-Chef (validates, merges), Commis (execute).
  Clients order via a Menu (PERT), the kitchen delivers Plats (merged features).
  Creates tmuxinator config, prompts, shared-state memory, quality gates, and permissions.
  Uses Claude Code Agent Teams (TeamCreate + SendMessage) with visible tmux panes.
  Use when the user wants to parallelize work across multiple Claude Code agents, orchestrate
  a multi-agent sprint, or says 'boss', 'multi-agent', 'orchestrate', 'parallelize agents',
  'team of claudes', 'sprint multi-agent', 'brigade', 'conductor', 'chef'.
  Also triggers on 'swarm', 'agent team', 'workers paralleles', 'PERT multi-agent'.
argument-hint: "[project-path-or-name]"
context: fork
agent: general-purpose
allowed-tools:
  - Read
  - Write
  - Edit
  - Grep
  - Glob
  - Bash
  - Agent
---

> **Optimization:** Heavy content lives in `references/`. Load on demand.

> **Language rule:** Detect the project's primary language. Output in that language.

> **Gotchas:** Read `../../gotchas.md` AND `references/gotchas-boss.md` before producing output.

# CLI Forge Boss — Brigade de Cuisine

> *"On ne cuisine pas seul. On cuisine en brigade."* — Auguste Escoffier

## La Brigade

```
┌─────────────────────────────────────────────────────┐
│                      CLIENT                          │
│  (utilisateur, ticket, roadmap)                      │
│                                                      │
│  Commande des plats via le MENU (PERT)               │
│  "Je veux: decision tokens + E2E + sokolsky"         │
└──────────────────────┬──────────────────────────────┘
                       │ commande
┌──────────────────────▼──────────────────────────────┐
│                  CHEF DE CUISINE                     │
│  (planifie, decide, orchestre)                       │
│                                                      │
│  - Lit le menu (PERT / roadmap)                      │
│  - Repartit les plats entre les commis               │
│  - Decide de l'ordre d'envoi                         │
│  - Demande au sous-chef de gouter avant envoi        │
│  - Annonce : "Envoyez !" quand c'est pret            │
│  - NE CUISINE JAMAIS                                 │
└──────────────────────┬──────────────────────────────┘
                       │ SendMessage
┌──────────────────────▼──────────────────────────────┐
│                   SOUS-CHEF                          │
│  (goute, valide, envoie)                             │
│                                                      │
│  - Goute les plats (quality gates /cli-audit-*)      │
│  - Controle le dressage (merge + CI)                 │
│  - Renvoie en cuisine si pas bon (gate failed)       │
│  - Envoie au pass (push + CI verte)                  │
│  - Met a jour le carnet (shared-state.md)            │
│  - Tourne en bypassPermissions (zero blocage)        │
└──────────────────────┬──────────────────────────────┘
                       │ SendMessage
┌──────────────────────▼──────────────────────────────┐
│              COMMIS (x N)                            │
│  (cuisinent, preparent, dressent)                    │
│                                                      │
│  - Chacun a son poste (worktree / projet)            │
│  - Preparent leur plat (code, test, commit)          │
│  - Annoncent "Pret !" au sous-chef                   │
│  - NE DECIDENT PAS de l'envoi                        │
└─────────────────────────────────────────────────────┘
```

### Le vocabulaire de la brigade

| Brigade | Code | Exemple |
|---------|------|---------|
| Menu | PERT / roadmap | "Sprint S3 : 4 plats" |
| Commande | Ticket / tache | "feat/hit-decision-tokens" |
| Plat | Feature merged + CI verte | PR #88 merged |
| Mise en place | shared-state.md "En cours" | Worker ecrit ses fichiers cibles |
| Cuisson | Coding + tests | Worker code dans son worktree |
| Dressage | Commit + cleanup | cargo fmt, clippy clean |
| Gouter | Quality gates | /cli-audit-code, /cli-audit-drift |
| Pass | CI pipeline | gh run watch |
| Envoi | Merge + feu vert | "Envoyez ! feat/x merge." |
| Renvoi | Gate failed | "Renvoie ! CQI trop bas." |
| Coup de feu | Phase parallele | 4 commis en meme temps |
| Service | Sprint complet | Phase 0 → Phase N → rapport |

### Flux de communication

```
Commis termine la cuisson
  → "Pret !" au Sous-Chef (SendMessage)
  → Sous-Chef goute (quality gates)
  → SI bon :
      → Sous-Chef dresse et envoie (merge + CI)
      → Sous-Chef au Chef : "Plat envoye, table servie"
      → Chef aux commis dependants : "Envoyez le suivant !"
  → SI pas bon :
      → Sous-Chef au Commis : "Renvoi ! Trop de sel dans fn X"
      → Sous-Chef au Chef : "Renvoi sur plat Y, commis corrige"
```

## Input

`$ARGUMENTS` is the target project path or name.

- If a path: analyze the project, generate config for it
- If a name: create a new project directory and generate config
- If empty: use the current working directory

## Phase 0 — Mise en place

Before generating anything, understand the project:

### 0.1 — Detect project type

| Signal | Type | Build tools | Test command | Lint command |
|--------|------|-------------|--------------|--------------|
| `Cargo.toml` | Rust | `cargo` | `cargo test` | `cargo fmt && cargo clippy` |
| `go.mod` | Go | `go` | `go test ./...` | `golangci-lint run` |
| `package.json` | JS/TS | `npm`, `npx` | `npm test` | `npm run lint` |
| `pyproject.toml` / `setup.py` | Python | `python3`, `pip`, `uv` | `pytest` | `ruff check` |
| `*.tf` / `terragrunt.hcl` | Terraform | `terraform`, `tofu` | `terraform plan` | `terraform validate` |
| `Chart.yaml` / `helmfile.yaml` | Helm | `helm` | `helm template` | `helm lint` |
| `docker-compose*.yml` | Docker | `docker`, `podman` | N/A | `docker compose config` |
| `flake.nix` | Nix | `nix` | `nix flake check` | `nix flake check` |
| Multiple of above | Monorepo | union | per-workspace | per-workspace |

If no manifest found → ask the user: "What type of project is this? (Rust, Go, JS/TS, Python, Infra, Other)"

Use the detected type to:
- Filter build tool permissions in `settings.local.json` (only include relevant tools)
- Set correct build/test/lint commands in commis prompts
- Choose appropriate quality gates

### 0.2 — Read the project

1. **Read the project** — CLAUDE.md, README, manifest file, src/ structure
2. **Read git state** — branches, worktrees, CI config
3. **Identify the menu** — what needs to be done? Check issues, roadmap, design docs

### 0.3 — Check prerequisites

| Tool | Check | Install (Fedora/RHEL) | Install (macOS) | Install (Debian/Ubuntu) | Fallback |
|------|-------|-----------------------|-----------------|-------------------------|----------|
| tmux | `which tmux` | `sudo dnf install tmux` | `brew install tmux` | `sudo apt install tmux` | None — required |
| tmuxinator | `which tmuxinator` | `gem install tmuxinator` | `gem install tmuxinator` | `gem install tmuxinator` | Raw tmux script (manual pane setup) |
| claude | `which claude` | `npm i -g @anthropic-ai/claude-code` | `npm i -g @anthropic-ai/claude-code` | `npm i -g @anthropic-ai/claude-code` | None — required |
| gh | `which gh` | `sudo dnf install gh` | `brew install gh` | `sudo apt install gh` | Manual CI checks |

```bash
# Check all prerequisites
for tool in tmux tmuxinator claude gh; do
  which "$tool" >/dev/null 2>&1 || echo "MISSING: $tool"
done
grep -q "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS" ~/.claude/settings.json 2>/dev/null || echo "AGENT_TEAMS_NOT_ENABLED"
```

If a tool is missing → **show the install command for the detected OS** and stop.
If Agent Teams not enabled → enable it.
If `teammateMode` not set → add `"teammateMode": "tmux"` in `~/.claude.json`.

### 0.4 — Choose brigade size (Mitosis)

Scale the brigade to the workload. Never deploy 5 commis for 2 tasks.

| Signal | Tier | Brigade | Tmuxinator | Quality gates |
|--------|------|---------|------------|---------------|
| 1-2 tasks, single repo | **S** | 1 commis, no sous-chef | No tmuxinator — single worktree, direct orchestration | `/cli-audit-code` only |
| 3-4 tasks, single repo | **M** | 2-3 commis + sous-chef | Standard config | Minimum viable profile |
| 5+ tasks or multi-repo | **L** | 4-5 commis + sous-chef | Full config + side-project panes | Standard or Complete profile |

For tier **S**: skip tmuxinator entirely. Use a single `claude` invocation with `--append-system-prompt` and one worktree. The chef handles quality gates directly.

## Phase 1 — Le menu

Ask these questions (skip if clear from context):

1. **Quels plats ?** (the tasks / features)
2. **Combien de commis ?** (default: 3, max: 5)
3. **Ordre d'envoi ?** (dependencies between tasks)
4. **Plats hors carte ?** (side projects / standalone repos)
5. **Quel niveau de degustation ?** (which `/cli-audit-*` quality gates)
6. **Branche de service ?** (develop, main, etc.)

## Phase 2 — La brigade s'installe

Read `references/templates.md` for templates. Generate:

### 2.1 — Le carnet de cuisine (`{project}/.claude/shared-state.md`)

Read `references/shared-state-template.md` and customize.

### 2.2 — Les consignes du Chef (`{project}/.claude/prompts/chef-{session}.md`)

Read `references/conductor-prompt-template.md`. The Chef:
- Creates the team (TeamCreate)
- Spawns Sous-Chef + N Commis (Agent tool)
- Sends the menu (PERT) to Sous-Chef
- Receives "plat envoye" from Sous-Chef
- Announces "envoyez !" to dependent commis
- Produces the rapport de service

### 2.3 — Les consignes du Sous-Chef (embedded)

Spawned by Chef. The Sous-Chef:
- Executes quality gates (/cli-audit-*)
- Merges in gate pane (tmux send-keys)
- Watches CI (gh run watch)
- Updates shared-state.md
- Reports to Chef

**CRITICAL:** Sous-Chef runs `mode: "bypassPermissions"` — il ne demande jamais.

### 2.4 — Les fiches de poste des Commis (embedded)

Each commis prompt includes:
- Shared state path (absolute)
- Mission (le plat a preparer)
- Steps (la recette)
- SendMessage au Sous-Chef quand pret (NOT au Chef)

### 2.5 — Tmuxinator (`~/.config/tmuxinator/{session}.yml`)

Read `references/tmuxinator-template.md`.

```yaml
# Chef — ALL these flags required (see gotchas)
- claude --dangerously-skip-permissions --permission-mode bypassPermissions --teammate-mode tmux --append-system-prompt "$(cat {project}/.claude/prompts/chef-{session}.md)"
```

### 2.6 — Permissions (`{project}/.claude/settings.local.json`)

Read `references/permissions-template.md`.

### 2.7 — Gotchas

**ALWAYS** include gotchas from `references/gotchas-boss.md` in the chef prompt.

## Phase 3 — Verification avant service

1. Check tmuxinator syntax: `tmuxinator doctor`
2. Verify all paths exist
3. Verify branches don't conflict
4. Verify permissions cover all operations
5. Count commis — warn if > 5

## Phase 4 — Bon appétit

Present to the user:
1. La brigade (diagram)
2. Le menu (PERT)
3. Files generated
4. How to launch: `tmuxinator start {session}`
5. Gotchas to watch for

## Reference files

| File | Content |
|------|---------|
| `references/gotchas-boss.md` | 14 known pitfalls and fixes |
| `references/templates.md` | Index of all templates |
| `references/shared-state-template.md` | Le carnet de cuisine |
| `references/conductor-prompt-template.md` | Consignes Chef + Sous-Chef + Commis |
| `references/tmuxinator-template.md` | Tmuxinator YAML |
| `references/permissions-template.md` | settings.local.json |
| `references/quality-gates.md` | La carte des degustations (audit skills) |

## Integration with other cli-* skills

| Skill | Relation |
|-------|----------|
| `/cli-audit-tangle` | Phase 0 — identifier les couplages pour assigner les modules aux commis sans conflits |
| `/cli-audit-code` | Quality gate par merge — le sous-chef l'execute avant chaque merge |
| `/cli-audit-drift` | Quality gate par merge — verifie que le code respecte les contrats |
| `/cli-audit-test` | Quality gate par merge — verifie la pyramide de tests |
| `/cli-audit-sync` | Quality gate post-merge — verifie la coherence docs/code |
| `/cli-cycle` | Fin de service — scorecard globale du sprint |
| `/cli-forge-pipeline` | Optimiser le pipeline CI que les commis vont declencher |
| `/cli-forge-tree` | Valider la structure du projet avant d'assigner les worktrees |
