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

> **Language rule:** Detect the project's primary language (from README, comments, docs, commit messages). Output ALL generated files (prompts, shared-state, tmuxinator comments) in that language. If the project is bilingual, ask the user which language to use before proceeding. The Brigade vocabulary (Menu, Commis, Sous-Chef, etc.) stays in French regardless — it's the pattern's terminology, not the output language.

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
│              3 SOUS-CHEFS VOTANTS                    │
│  (goutent, jugent — quorum 2/3)                      │
│                                                      │
│  sous-chef-scope   : le plat est dans le poste?      │
│  sous-chef-secu    : pas de verre pile dans le plat? │
│  sous-chef-qualite : le gout est bon?                │
│                                                      │
│  2/3 APPROVE → passe                                 │
│  1 DENY → renvoi en cuisine                          │
│  1 ESCALATE → appel au patron (humain, ~5% des cas)  │
├──────────────────────────────────────────────────────┤
│              SOUS-CHEF MERGE                         │
│  (dresse, envoie au pass)                            │
│                                                      │
│  - Quality gates (/cli-audit-*)                      │
│  - Merge + CI (tmux send-keys gate)                  │
│  - Met a jour le carnet (shared-state.md)            │
│  - Tourne en bypassPermissions                       │
└──────────────────────┬──────────────────────────────┘
                       │ SendMessage
┌──────────────────────▼──────────────────────────────┐
│              COMMIS (x N)                            │
│  (cuisinent, preparent, dressent)                    │
│                                                      │
│  - Chacun a son poste (worktree / projet)            │
│  - Preparent leur plat (code, test, commit)          │
│  - Annoncent "Pret !" au sous-chef-merge             │
│  - NE DECIDENT PAS de l'envoi                        │
└─────────────────────────────────────────────────────┘
```

### Quand le patron (humain) intervient

**~5% des cas.** L'humain est appele uniquement quand un Sous-Chef vote ESCALATE :

| Cas | Qui escalade | Pourquoi |
|---|---|---|
| Edit sur `.github/workflows/` | sous-chef-scope | CI = impact global |
| Nouvelle dep dans Cargo.toml | sous-chef-secu | Supply chain risk |
| Suppression de tests | sous-chef-secu | Jamais auto-approve |
| Diff > 200 lignes | sous-chef-qualite | Trop gros pour juger vite |
| Fichier d'un autre worker | sous-chef-scope | Conflit potentiel |

Tout le reste passe sans intervention humaine.

### Le vocabulaire de la brigade

| Brigade | Code | Exemple |
|---------|------|---------|
| **Carte du jour** | Revue zones sensibles (debut de sprint) | "ci.yml passe en 3/3, log_backend reste en 2/3" |
| **Marche** | Inventaire du projet (git log, PRs, incidents) | "3 DENY sur ci.yml au sprint S3" |
| **Produit frais** | Zone normale (2/3) sans incident | "src/features/ : 0 DENY en 3 sprints" |
| **Produit sensible** | Zone 3/3 unanimite requise | "Cargo.toml : advisory rustls-pemfile" |
| **Produit retire** | Hallucination passee → cas de test | "Auto-approve aveugle → G16" |
| **Nouvelle recette** | Nouveau module a classifier | "log_backend → normal, self_tuning → sensible" |
| Menu | PERT / roadmap | "Sprint S3 : 4 plats" |
| Commande | Ticket / tache | "feat/hit-decision-tokens" |
| Plat | Feature merged + CI verte | PR #88 merged |
| Mise en place | shared-state.md "En cours" | Worker ecrit ses fichiers cibles |
| Cuisson | Coding + tests | Worker code dans son worktree |
| Dressage | Commit + cleanup | cargo fmt, clippy clean |
| Gouter | Vote quorum des Sous-Chefs | "2/3 APPROVE (normal) ou 3/3 (sensible)" |
| Pass | CI pipeline | gh run watch |
| Envoi | Merge + feu vert | "Envoyez ! feat/x merge." |
| Renvoi | Gate failed ou quorum DENY | "Renvoie ! Le Sous-Chef Secu propose: ..." |
| Appel au patron | ESCALATE (< 2%) | "Patron, edit sur ci.yml, 3 rounds sans consensus" |
| Coup de feu | Phase parallele | 4 commis en meme temps |
| Service | Sprint complet | Phase 0 → Phase N → rapport |

### Flux de communication

```
Commis edite un fichier
  → Chef envoie le diff aux 3 Sous-Chefs votants (parallele)
  → 2/3 APPROVE → passe automatiquement (zero humain)
  → 1 DENY → renvoi au commis avec la raison
  → 1 ESCALATE → Chef notifie le patron (humain)

Commis termine la cuisson
  → "Pret !" au Sous-Chef Merge (SendMessage)
  → Sous-Chef Merge goute (quality gates)
  → SI bon :
      → Sous-Chef Merge dresse et envoie (merge + CI)
      → Sous-Chef Merge au Chef : "Plat envoye, table servie"
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

1. **Read the project** — README, CONTRIBUTING.md, docs/explanation/architecture.md, manifest file, src/ structure
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

**Auto-fix configuration (G19, G20, G22) — do ALL of these automatically, don't ask :**

1. `~/.claude/settings.json` : ensure `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` + `skipDangerousModePermissionPrompt=true`
2. `~/.claude.json` : ensure `"teammateMode": "tmux"`
3. `{project}/.claude/settings.local.json` : ensure all permissions (build tools, shared-state, external repos, Obsidian vault, Team tools)
4. Tmuxinator `on_project_start` : ensure auto-kick message with `sleep 12 && tmux send-keys ... &` (G20)
5. If project has an Obsidian vault : add Edit/Read permissions for the vault path (G22)
6. Warn user : **do NOT run `brew upgrade claude-code` while the session is running** (G18)

### 0.4 — Choose brigade size (Mitosis)

Scale the brigade to the workload. Never deploy 5 commis for 2 tasks.

| Signal | Tier | Brigade | Sous-Chefs | Quality gates |
|--------|------|---------|------------|---------------|
| 1-2 tasks, single repo | **S** | 1 commis, Chef fait tout | 0 — Chef valide lui-meme | `/cli-audit-code` only |
| 3-4 tasks, single repo | **M** | 2-3 commis | 3 sous-chefs (scope, secu, qualite) + 1 merge | Minimum viable |
| 5+ tasks or multi-repo | **L** | 4-5 commis | 3 sous-chefs + 1 merge | Standard |
| 10+ tasks, monorepo, regulated | **XL** | 5+ commis | Grappes de sous-chefs (voir ci-dessous) | Complet |

For tier **S**: skip tmuxinator entirely. Use a single `claude` invocation with `--append-system-prompt` and one worktree. The chef handles quality gates directly.

### Tier XL — Grappes de Sous-Chefs (Brigades de parties)

Pour les gros projets (monorepo, multi-domaines, reglemente), chaque domaine a sa propre grappe de 3 sous-chefs. Chaque grappe vote en interne (2/3), puis les grappes votent entre elles (2/3 normal, 3/3 sensible).

```
Chef de Cuisine
  │
  ├── Grappe SECURITE (3 sous-chefs) — vote interne 2/3
  │   ├── sous-chef-secrets     (DLP, credentials, env vars)
  │   ├── sous-chef-deps        (supply chain, advisories, licences)
  │   └── sous-chef-perimeter   (scope, acces, permissions fichiers)
  │   → Produit 1 avis consolide : APPROVE / DENY+solution
  │
  ├── Grappe QUALITE (3 sous-chefs) — vote interne 2/3
  │   ├── sous-chef-coherence   (mission, design, architecture)
  │   ├── sous-chef-tests       (coverage, pyramide, regression)
  │   └── sous-chef-proprete    (code smells, dead code, naming)
  │   → Produit 1 avis consolide
  │
  └── Grappe OPS (3 sous-chefs) — vote interne 2/3
      ├── sous-chef-ci          (pipelines, workflows, matrix)
      ├── sous-chef-deploy      (infra, docker, helm)
      └── sous-chef-perf        (benchmarks, timeouts, resources)
      → Produit 1 avis consolide

Vote inter-grappes :
  Zone normale : 2/3 grappes APPROVE → passe
  Zone sensible : 3/3 grappes APPROVE → passe
  DENY : la grappe qui DENY propose une solution
  Rounds de resolution : identiques au tier M
```

Le tier XL c'est **9 sous-chefs + 1 merge = 10 agents de validation**. A utiliser uniquement quand le cout des erreurs justifie le cout des agents (reglemente, defense, finance).

**Logique de choix (le Chef decide au Phase 0) :**

```
Le Chef detecte le tier au demarrage :
  - Nombre de taches → S/M/L
  - Multi-repo → L minimum
  - Fichier CONTRACTS.md ou compliance/ → XL
  - Mention "regulated", "defense", "finance" dans README/CONTRIBUTING.md → XL
  - L'utilisateur peut forcer : /cli-forge-boss --tier XL
```

### 0.5 — Build coupling matrix

Read `references/conflict-resolution.md`. Before assigning tasks:

1. Run `/cli-audit-tangle` on the project
2. Build the coupling matrix (file × task: R/W)
3. If W/W conflict on same file → assign to same commis OR sequence in PERT
4. Mark "hot files" (Cargo.toml, ci.yml, mod.rs) → managed by Sous-Chef only

### 0.6 — Identify exploration candidates

Read `references/parallel-exploration.md`. For each task, the Chef asks:

- Is there a non-trivial architecture choice? → mark as EXPLORATION
- Is there a complex bug with unknown root cause? → mark as HYPOTHESES
- Did the user ask to "compare approaches"? → mark as EXPLORATION

EXPLORATION tasks use 2-3 commis on the same problem with different approaches. The Sous-Chef runs the comparison grid and the Chef picks the winner. **Confirm with user before launching** (2-3x token cost).

## Phase 1 — Le menu (auto-detection, NE PAS DEMANDER)

**REGLE LANGUE : `git log --oneline -10` + `head -20 README.md`. Si le projet est en francais → TOUS les fichiers generes (prompts, shared-state, tmuxinator comments, rapports) sont en francais. Si en anglais → anglais. Le vocabulaire de la brigade (Menu, Commis, Sous-Chef) reste en francais quel que soit le projet.**

**REGLE COMMITS : Les commis et le Sous-Chef Merge utilisent `cli-git-conventional` pour TOUS les commits. Ghostwriter style, zero marqueur AI, langue du projet.**

**REGLE : Ne pose AUCUNE question si la reponse est dans le projet.**
Lis TOUT avant de demander quoi que ce soit :

1. **Quels plats ?** → Lire dans cet ordre jusqu'a trouver :
   - `{project}/.claude/shared-state.md` section "Backlog" (travail du sprint precedent)
   - Roadmap Obsidian (si vault specifie dans shared-state)
   - `gh issue list` (issues ouvertes)
   - `git log --oneline -20` (travail recent, ce qui manque)
   - README.md section TODO/roadmap
   - CONTRIBUTING.md section roadmap/next steps
   Si TOUJOURS rien → demander

2. **Combien de commis ?** → Compter les taches trouvees en 1, appliquer le tier (S/M/L/XL). Ne pas demander.

3. **Ordre d'envoi ?** → Deduire des dependances :
   - `cli-audit-tangle` (couplages = sequence obligatoire)
   - Si tache B touche un fichier cree par tache A → B depend de A
   - Si pas de dependance → parallele
   Ne pas demander.

4. **Plats hors carte ?** → Scanner :
   - `ls {workspace}/` pour les repos voisins
   - shared-state.md section "Contexte partage" (liens vers d'autres repos)
   - README.md / CONTRIBUTING.md (references a d'autres projets)
   Ne pas demander.

5. **Quel niveau de degustation ?** → Deduire du tier :
   - S → `/cli-audit-code` only
   - M → minimum viable (code + drift)
   - L → standard (code + drift + test + sync)
   - XL → complet
   Ne pas demander.

6. **Branche de service ?** → `git branch -a | head` + CONTRIBUTING.md. C'est toujours ecrit. Ne pas demander.

**Ne demander a l'utilisateur QUE si :**
- Aucune source ne contient l'information (pas de roadmap, pas d'issues, pas de backlog)
- Il y a une ambiguite critique (2 roadmaps contradictoires)
- Le tier XL est detecte (confirmation cout tokens)

Presenter le menu deduit et demander confirmation : "Voici ce que je vais lancer. OK ?"

## Phase 2 — La brigade s'installe

Read `references/templates.md` for templates. Generate:

### 2.1 — Le carnet de cuisine (`{project}/.claude/shared-state.md`)

Read `references/shared-state-template.md` and customize.

### 2.2 — Les consignes du Chef (`{project}/.claude/prompts/chef-{session}.md`)

**OBLIGATION ABSOLUE : Read `references/conductor-prompt-template.md` BEFORE generating the prompt.**
**Le prompt genere DOIT contenir les 3 Sous-Chefs votants + le Sous-Chef Merge.**
**Si le prompt ne contient pas "sous-chef-scope", "sous-chef-secu", "sous-chef-qualite", "sous-chef-merge" → le prompt est INVALIDE.**

Le prompt du Chef DOIT inclure dans cet ordre :
1. TeamCreate
2. **Spawn des 3 Sous-Chefs votants** (scope, secu, qualite) — copier VERBATIM depuis le template
3. **Spawn du Sous-Chef Merge** — copier VERBATIM depuis le template
4. **Protocole de vote avec rounds de resolution** — copier VERBATIM depuis le template
5. Spawn des N Commis
6. PERT
7. Shutdown protocol (avec revue zones sensibles)

Le Chef :
- Creates the team (TeamCreate)
- Spawns 3 Sous-Chefs votants + 1 Sous-Chef Merge + N Commis
- Les permissions des Commis passent par les 3 Sous-Chefs (quorum 2/3 normal, 3/3 sensible)
- Le Sous-Chef Merge gere les quality gates, merges, CI
- Produces the rapport de service

### 2.3 — Les 3 Sous-Chefs votants (OBLIGATOIRE — embedded dans le prompt Chef)

**COPIER les prompts des 3 sous-chefs depuis `references/conductor-prompt-template.md` section "Spawn les 3 Sous-Chefs".**

Chacun vote independamment : APPROVE / DENY+solution / CONCERN+solution.
Rounds de resolution si pas de consensus. Escalade humaine < 2%.

### 2.4 — Le Sous-Chef Merge (OBLIGATOIRE — embedded dans le prompt Chef)

**COPIER le prompt depuis `references/conductor-prompt-template.md` section "Spawn le Sous-Chef Merge".**

Gere les quality gates, merges, CI. Ne vote pas sur les permissions.

### 2.5 — Les fiches de poste des Commis (embedded)

Each commis prompt includes:
- Shared state path (absolute)
- Mission (le plat a preparer)
- Steps (la recette)
- SendMessage au Sous-Chef Merge quand pret (NOT au Chef)
- Les permissions passent par le quorum des 3 Sous-Chefs (le commis ne le sait pas)

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

## Phase 5 — Lancer le Sous-Chef automatique

Apres que l'utilisateur lance `tmuxinator start`, **lancer automatiquement** le Sous-Chef via `/loop` :

```
/loop 2m Sous-Chef check. Regarde le pane boss {session}:0.0 avec tmux capture-pane. Si permission en attente ("Do you want to make this edit"), lis le diff. Zones sensibles (NE PAS approuver, SKIP) : {zones_sensibles}. Zones normales : tout le reste → approuve avec tmux send-keys -t {session}:0.0 Enter. PUIS recheck immediatement (sleep 5 + capture-pane) — tant qu'il y a des permissions en file, continue a approuver. Arrete quand le boss est libre. Log chaque action.
```

Le Sous-Chef tourne dans la session de l'utilisateur (cette session, pas dans tmux). Il surveille le boss toutes les 2 minutes et :
- **APPROVE** les edits en zone normale (src/, tests/, shared-state.md)
- **SKIP** les edits en zone sensible (CI, deps, auth, security) — le boss reste bloque, l'utilisateur decidera a son retour
- **Log** chaque decision pour tracabilite

L'utilisateur peut partir. Le Sous-Chef veille.

Pour arreter : `CronDelete {job_id}`

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
| `references/conflict-resolution.md` | Arbre de decision, matrice de couplage, file locking, escalade |
| `references/parallel-exploration.md` | Hypotheses concurrentes, approches paralleles, grille de comparaison |

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
| `/cli-git-conventional` | **TOUJOURS** — tous les commits des commis et du Sous-Chef Merge passent par ghostwriter. Zero AI markers |
| `/cli-forge-doc` | Si quality gate detecte doc manquante — le Sous-Chef Merge declenche la generation |
| `/cli-forge-schema` | Si quality gate detecte diagramme manquant — generation Mermaid |
