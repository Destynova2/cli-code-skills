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
  'team of claudes', 'sprint multi-agent', 'brigade', 'chef'.
  Also triggers on 'swarm', 'agent team', 'parallel workers', 'PERT multi-agent'.
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

> **Language rule:** Skill instructions are written in English. When generating user-facing files (prompts, shared-state, tmuxinator comments, reports), detect the project's primary language (from README, comments, docs, commit messages) and produce those files in that language. If the project is bilingual, ask the user which language to use before proceeding. The Brigade vocabulary (Menu, Commis, Sous-Chef, Chef, Plat, Mise en place, etc.) stays in French regardless — it's the pattern's canonical terminology, not prose.

> **Gotchas:** Read `../../gotchas.md` AND `references/gotchas-boss.md` before producing output.

# CLI Forge Boss — Brigade de Cuisine

> *"On ne cuisine pas seul. On cuisine en brigade."* — Auguste Escoffier
> (*"You don't cook alone. You cook in a brigade."*)

## The Brigade

```
┌─────────────────────────────────────────────────────┐
│                      CLIENT                          │
│  (user, ticket, roadmap)                             │
│                                                      │
│  Orders plats via the MENU (PERT)                    │
│  "I want: decision tokens + E2E + sokolsky"          │
└──────────────────────┬──────────────────────────────┘
                       │ order
┌──────────────────────▼──────────────────────────────┐
│                  CHEF DE CUISINE                     │
│  (plans, decides, orchestrates)                      │
│                                                      │
│  - Reads the menu (PERT / roadmap)                   │
│  - Assigns plats to the commis                       │
│  - Decides the order in which to send them out       │
│  - Asks the Sous-Chef to taste before sending        │
│  - Calls "Envoyez !" when it's ready                 │
│  - NEVER COOKS                                       │
└──────────────────────┬──────────────────────────────┘
                       │ SendMessage
┌──────────────────────▼──────────────────────────────┐
│              3 VOTING SOUS-CHEFS                     │
│  (taste, judge — quorum 2/3)                         │
│                                                      │
│  sous-chef-scope   : is the plat in your station?    │
│  sous-chef-secu    : any broken glass in the plat?   │
│  sous-chef-qualite : does it taste right?            │
│                                                      │
│  2/3 APPROVE → passes                                │
│  1 DENY → sent back to the kitchen                   │
│  1 ESCALATE → call the patron (human, ~5% of cases)  │
├──────────────────────────────────────────────────────┤
│              SOUS-CHEF MERGE                         │
│  (plates up, sends to the pass)                      │
│                                                      │
│  - Quality gates (/cli-audit-*)                      │
│  - Merge + CI (tmux send-keys gate)                  │
│  - Updates the carnet (shared-state.md)              │
│  - Runs in bypassPermissions                         │
└──────────────────────┬──────────────────────────────┘
                       │ SendMessage
┌──────────────────────▼──────────────────────────────┐
│              COMMIS (x N)                            │
│  (cook, prep, plate)                                 │
│                                                      │
│  - Each has their own station (worktree / project)   │
│  - Prepare their plat (code, test, commit)           │
│  - Announce "Pret !" to the sous-chef          │
│  - DO NOT DECIDE when to send out                    │
└─────────────────────────────────────────────────────┘
```

### When the patron (human) steps in

**~5% of cases.** The human is called only when a Sous-Chef votes ESCALATE:

| Case | Who escalates | Why |
|---|---|---|
| Edit on `.github/workflows/` | sous-chef-scope | CI = global impact |
| New dep in Cargo.toml | sous-chef-secu | Supply chain risk |
| Test removal | sous-chef-secu | Never auto-approve |
| Diff > 200 lines | sous-chef-qualite | Too big to judge quickly |
| File from another worker | sous-chef-scope | Potential conflict |

Everything else passes without human intervention.

### Brigade vocabulary

| Brigade term | Meaning | Example |
|---|---|---|
| **Carte du jour** | Review of sensitive zones (sprint start) | "ci.yml moves to 3/3, log_backend stays 2/3" |
| **Marche** | Project inventory (git log, PRs, incidents) | "3 DENYs on ci.yml during sprint S3" |
| **Produit frais** | Normal zone (2/3) with no incidents | "src/features/ : 0 DENY in 3 sprints" |
| **Produit sensible** | 3/3 unanimity required | "Cargo.toml: rustls-pemfile advisory" |
| **Produit retire** | Past hallucination → now a test case | "Blind auto-approve → G16" |
| **Nouvelle recette** | New module to classify | "log_backend → normal, self_tuning → sensitive" |
| Menu | PERT / roadmap | "Sprint S3: 4 plats" |
| Commande | Ticket / task | "feat/hit-decision-tokens" |
| Plat | Feature merged + CI green | PR #88 merged |
| Mise en place | shared-state.md "In progress" | Worker writes its target files |
| Cuisson | Coding + tests | Worker codes in its worktree |
| Dressage | Commit + cleanup | cargo fmt, clippy clean |
| Gouter | Quorum vote by the Sous-Chefs | "2/3 APPROVE (normal) or 3/3 (sensitive)" |
| Pass | CI pipeline | gh run watch |
| Envoi | Merge + green light | "Envoyez! feat/x merged." |
| Renvoi | Gate failed or quorum DENY | "Renvoie! Sous-Chef Secu proposes: ..." |
| Appel au patron | ESCALATE (< 2%) | "Patron, edit on ci.yml, 3 rounds without consensus" |
| Coup de feu | Parallel phase | 4 commis at the same time |
| Service | Full sprint | Phase 0 → Phase N → report |

### Communication flow

```
Commis edits a file
  → Chef sends the diff to the 3 voting Sous-Chefs (parallel)
  → 2/3 APPROVE → passes automatically (zero human)
  → 1 DENY → sent back to the commis with a reason
  → 1 ESCALATE → Chef notifies the patron (human)

Commis finishes cooking
  → "Pret !" to the Sous-Chef (SendMessage)
  → Sous-Chef tastes (quality gates)
  → IF good:
      → Sous-Chef plates up and sends (merge + CI)
      → Sous-Chef to Chef: "Plat sent, table served"
      → Chef to dependent commis: "Envoyez the next one!"
  → IF not good:
      → Sous-Chef to Commis: "Renvoi! Too much salt in fn X"
      → Sous-Chef to Chef: "Renvoi on plat Y, commis is fixing it"
```

## Input

`$ARGUMENTS` is the target project path or name.

- If a path: analyze the project, generate config for it
- If a name: create a new project directory and generate config
- If empty: use the current working directory

## Phase 0 — Mise en place

**First, check whether a sprint is currently running or paused:**

Read `references/sprint-persistence.md`. Check `.claude/sprint-history/current/`:
- If a sprint is **PAUSED** → offer: Resume / Rewind / Fresh / Abandon
- If a sprint is **DONE** → read the gotchas-learned and sensitive zones for the new sprint
- If no history → fresh sprint

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
| tmuxinator | `which tmuxinator` | `brew install tmuxinator` (preferred) or `gem install tmuxinator` | `brew install tmuxinator` | `gem install tmuxinator` | **Raw tmux script** — see `references/raw-tmux-fallback.md` |
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

**Exception: missing tmuxinator is NOT a blocker.** If Ruby/gem is not available (air-gapped env, immutable OS like Bluefin/Silverblue), generate the raw-tmux fallback instead:
- Read `references/raw-tmux-fallback.md`
- Generate `{project}/.claude/scripts/boss-{session}.sh` (chmod +x)
- Document in the Phase 4 report: "tmuxinator missing → raw-tmux fallback generated"
- Launch with `bash {project}/.claude/scripts/boss-{session}.sh` instead of `tmuxinator start {session}`

**Auto-fix configuration (G19, G20, G22) — do ALL of these automatically, don't ask:**

1. `~/.claude/settings.json`: ensure `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` + `skipDangerousModePermissionPrompt=true`
2. `~/.claude.json`: ensure `"teammateMode": "tmux"`
3. `{project}/.claude/settings.local.json`: ensure all permissions (build tools, shared-state, external repos, Obsidian vault, Team tools)
4. Tmuxinator `on_project_start`: ensure auto-kick message with `sleep 12 && tmux send-keys ... &` (G20)
5. If project has an Obsidian vault: add Edit/Read permissions for the vault path (G22)
6. Warn user: **do NOT run `brew upgrade claude-code` while the session is running** (G18)

### 0.4 — Choose model and brigade size (Mitosis)

Read `references/simplified-model.md`. Two models exist — pick based on tier:

| Signal | Tier | Model | Agents | Quality gates |
|--------|------|--------|--------|---------------|
| 1-2 tasks, single repo | **S** | No brigade | 1 commis alone | Auto-check (proofreading) |
| 3-4 tasks, single repo | **M** | **Stigmergy** | 1 sous-chef (3 lenses) + N commis | Adaptive (DNA repair) |
| 5+ tasks or multi-repo | **L** | **Stigmergy** | 1 sous-chef (3 lenses) + N commis | Adaptive (DNA repair) |
| 10+ tasks, monorepo, regulated | **XL** | **Full brigade** | Chef + sous-chef clusters + N commis | Full (vote, audit trail) |

**Tiers S/M/L**: the Chef generates files in Phase 0 and then disappears. Commis self-organize via shared-state.md (Boids + stigmergy). The single Sous-Chef reviews with 3 lenses. Commis that fail self-terminate (apoptosis).

**Tier XL**: the full brigade is kept for regulatory compliance (vote audit trail, decision traceability).

For tier **S**: skip tmuxinator entirely. Use a single `claude` invocation with `--append-system-prompt` and one worktree.

### Tier XL — Sous-Chef clusters (Brigades de parties)

For large projects (monorepos, multi-domain, regulated), each domain has its own cluster of 3 sous-chefs. Each cluster votes internally (2/3), then the clusters vote among themselves (2/3 normal, 3/3 sensitive).

```
Chef de Cuisine
  │
  ├── SECURITY cluster (3 sous-chefs) — internal vote 2/3
  │   ├── sous-chef-secrets     (DLP, credentials, env vars)
  │   ├── sous-chef-deps        (supply chain, advisories, licenses)
  │   └── sous-chef-perimeter   (scope, access, file permissions)
  │   → Produces 1 consolidated verdict: APPROVE / DENY+solution
  │
  ├── QUALITY cluster (3 sous-chefs) — internal vote 2/3
  │   ├── sous-chef-coherence   (mission, design, architecture)
  │   ├── sous-chef-tests       (coverage, pyramid, regression)
  │   └── sous-chef-proprete    (code smells, dead code, naming)
  │   → Produces 1 consolidated verdict
  │
  └── OPS cluster (3 sous-chefs) — internal vote 2/3
      ├── sous-chef-ci          (pipelines, workflows, matrix)
      ├── sous-chef-deploy      (infra, docker, helm)
      └── sous-chef-perf        (benchmarks, timeouts, resources)
      → Produces 1 consolidated verdict

Inter-cluster vote:
  Normal zone: 2/3 clusters APPROVE → passes
  Sensitive zone: 3/3 clusters APPROVE → passes
  DENY: the cluster that DENYs proposes a solution
  Resolution rounds: identical to tier M
```

Tier XL means **9 sous-chefs + 1 merge = 10 validation agents**. Use it only when the cost of errors justifies the cost of the agents (regulated, defense, finance).

**Selection logic (the Chef decides in Phase 0):**

```
The Chef detects the tier at startup:
  - Number of tasks → S/M/L
  - Multi-repo → L minimum
  - CONTRACTS.md file or compliance/ → XL
  - Mention of "regulated", "defense", "finance" in README/CONTRIBUTING.md → XL
  - User can force: /cli-forge-boss --tier XL
```

**Cost confirmation for tier XL (MANDATORY — auto-detect or flag):**

Tier XL costs roughly **5-8x** the tier L token spend (10 validation agents instead of ~2, each diff voted on by 9 sous-chefs instead of 3, audit trail generated every round).

Before spawning the XL brigade, **always** display and ask for confirmation:

```
=== TIER XL DETECTED ===
Reason: {detection_reason}  # e.g., "CONTRACTS.md present" or "--tier XL flag"
Brigade: Chef + 9 Sous-Chefs (3 clusters) + 1 Sous-Chef + N Commis
Estimated cost: ~5-8x tier L (each diff voted on by 9 agents instead of 3)
Estimated duration: ~2-3x tier L (inter-cluster resolution rounds)

Recommended justification:
  - Regulated project (defense, finance, healthcare)
  - Legal audit trail required
  - Cost of errors > cost of agents

Continue with XL? [y/N/downgrade-to-L]
```

If the user answers `downgrade-to-L`, switch to tier L and note in the Phase 4 report: "Tier XL detected but downgraded to L at user request".

**This confirmation is as mandatory as the Phase 0.6 confirmation for EXPLORATION.**

### 0.5 — Build coupling matrix

Read `references/conflict-resolution.md`. Before assigning tasks:

1. Run `/cli-audit-tangle` on the project
2. Build the coupling matrix (file × task: R/W)
3. If W/W conflict on the same file → assign to the same commis OR sequence in the PERT
4. Mark "hot files" (Cargo.toml, ci.yml, mod.rs) → managed by the Sous-Chef only

### 0.6 — Identify exploration candidates

Read `references/parallel-exploration.md`. For each task, the Chef asks:

- Is there a non-trivial architecture choice? → mark as EXPLORATION
- Is there a complex bug with unknown root cause? → mark as HYPOTHESES
- Did the user ask to "compare approaches"? → mark as EXPLORATION

EXPLORATION tasks use 2-3 commis on the same problem with different approaches. The Sous-Chef runs the comparison grid and the Chef picks the winner. **Confirm with user before launching** (2-3x token cost).

## Phase 1 — The menu (auto-detection, DO NOT ASK)

**LANGUAGE RULE: run `git log --oneline -10` + `head -20 README.md`. If the project is in French → ALL generated files (prompts, shared-state, tmuxinator comments, reports) are in French. If English → English. The Brigade vocabulary (Menu, Commis, Sous-Chef) stays in French regardless of the project language.**

**COMMITS RULE: commis and the Sous-Chef use `cli-git-conventional` for ALL commits. Ghostwriter style, zero AI marker, project language.**

**RULE: do NOT ask any question if the answer is in the project.**
Read EVERYTHING before asking anything:

1. **Which plats?** → Read in this order until found:
   - `{project}/.claude/shared-state.md` "Backlog" section (work from the previous sprint)
   - Obsidian roadmap (if vault is specified in shared-state)
   - `gh issue list` (open issues)
   - `git log --oneline -20` (recent work, what's missing)
   - README.md TODO/roadmap section
   - CONTRIBUTING.md roadmap/next steps section
   Only if NOTHING found → ask

2. **How many commis?** → Count the tasks found in step 1, apply the tier (S/M/L/XL). Don't ask.

3. **Sending order?** → Derive from dependencies:
   - `cli-audit-tangle` (couplings = required sequence)
   - If task B touches a file created by task A → B depends on A
   - Otherwise → parallel
   Don't ask.

4. **Plats off-menu?** → Scan:
   - `ls {workspace}/` for neighbouring repos
   - shared-state.md "Shared context" section (links to other repos)
   - README.md / CONTRIBUTING.md (references to other projects)
   Don't ask.

5. **What tasting level?** → Derive from the tier:
   - S → `/cli-audit-code` only
   - M → minimum viable (code + drift)
   - L → standard (code + drift + test + sync)
   - XL → complete
   Don't ask.

6. **Service branch?** → `git branch -a | head` + CONTRIBUTING.md. It's always written down. Don't ask.

**Only ask the user if:**
- No source contains the information (no roadmap, no issues, no backlog)
- There is a critical ambiguity (2 contradictory roadmaps)
- Tier XL is detected (token cost confirmation)

Present the deduced menu and ask for confirmation: "Here's what I'm going to launch. OK?"

## Phase 2 — The brigade sets up

Read `references/templates.md` for templates. Generate:

### 2.1 — The carnet de cuisine (`{project}/.claude/shared-state.md`)

Read `references/shared-state-template.md` and customize.

### 2.2 — The Chef's instructions (`{project}/.claude/prompts/chef-{session}.md`)

**ABSOLUTE REQUIREMENT: read `references/chef-prompt-template.md` BEFORE generating the prompt.**
**The generated prompt MUST contain the 3 voting Sous-Chefs + the Sous-Chef.**
**If the prompt does not contain "sous-chef-scope", "sous-chef-secu", "sous-chef-qualite", "sous-chef" → the prompt is INVALID.**

The Chef prompt MUST include, in this order:
1. TeamCreate
2. **Spawn the 3 voting Sous-Chefs** (scope, secu, qualite) — copy VERBATIM from the template
3. **Spawn the Sous-Chef** — copy VERBATIM from the template
4. **Voting protocol with resolution rounds** — copy VERBATIM from the template
5. Spawn the N Commis
6. PERT
7. Shutdown protocol (with sensitive-zone review)

The Chef:
- Creates the team (TeamCreate)
- Spawns 3 voting Sous-Chefs + 1 Sous-Chef + N Commis
- Commis permissions flow through the 3 Sous-Chefs (quorum 2/3 normal, 3/3 sensitive)
- The Sous-Chef handles quality gates, merges, CI
- Produces the service report

### 2.3 — The 3 voting Sous-Chefs (MANDATORY — embedded in the Chef prompt)

**COPY the prompts of the 3 sous-chefs from `references/chef-prompt-template.md` section "Spawn the 3 Sous-Chefs".**

Each votes independently: APPROVE / DENY+solution / CONCERN+solution.
Resolution rounds if no consensus. Human escalation < 2%.

### 2.4 — The Sous-Chef (MANDATORY — embedded in the Chef prompt)

**COPY the prompt from `references/chef-prompt-template.md` section "Spawn the Sous-Chef".**

Handles quality gates, merges, CI. Does not vote on permissions.

### 2.5 — Commis station sheets (embedded)

Each commis prompt includes:
- Shared state path (absolute)
- Mission (the plat to prepare)
- Steps (the recipe)
- SendMessage to the Sous-Chef when ready (NOT to the Chef)
- Permissions go through the quorum of the 3 Sous-Chefs (the commis does not know this)

### 2.5 — Tmuxinator (`~/.config/tmuxinator/{session}.yml`)

Read `references/tmuxinator-template.md`.

```yaml
# Chef — ALL these flags required (see gotchas)
- claude --dangerously-skip-permissions --permission-mode bypassPermissions --teammate-mode tmux --append-system-prompt "$(cat {project}/.claude/prompts/chef-{session}.md)"
```

### 2.6 — The contre-chef / ccheck (MANDATORY — G24)

**Generate `{project}/.claude/prompts/ccheck-{session}.md`.**

Read `references/ccheck-prompt-template.md` and customize with the project paths.

The ccheck is a dedicated Claude instance in its own tmux window that watches the Chef pane and auto-approves permissions in normal zones. **Without it, the Chef blocks on every worker edit and the brigade stalls.**

The ccheck:
- **APPROVES** edits in normal zones (src/, tests/, shared-state.md, docs/)
- **SKIPS** edits in sensitive zones (CI, deps, auth, secrets) — the Chef stays blocked, the user decides later
- **LOGS** every decision to `{project}/.claude/ccheck.log`
- **Re-reads** the sensitive zone list every iteration (so the Chef can add zones mid-sprint)

**This replaces the Phase 5 `/loop` approach** which was fragile (depended on the user's session staying open) and optional (it should never have been optional).

### 2.7 — Permissions (`{project}/.claude/settings.local.json`)

Read `references/permissions-template.md`.

### 2.7 — Gotchas

**ALWAYS** include gotchas from `references/gotchas-boss.md` in the chef prompt.

## Phase 3 — Check before service

1. Check tmuxinator syntax: `tmuxinator doctor`
2. Verify all paths exist
3. Verify branches don't conflict
4. Verify permissions cover all operations
5. Count commis — warn if > 5
6. **MANDATORY — ccheck validation (G24):**
   - `grep -q 'ccheck' ~/.config/tmuxinator/{session}.yml` — if missing, **STOP and add the ccheck window**
   - `test -f {project}/.claude/prompts/ccheck-{session}.md` — if missing, **STOP and generate the ccheck prompt**
   - If either check fails, do NOT proceed to Phase 4. Fix the generation first.
   - This is a hard gate, not a warning. The brigade WILL stall without the ccheck.

## Phase 4 — Bon appétit

Present to the user:
1. The brigade (diagram)
2. The menu (PERT)
3. Files generated
4. How to launch: `tmuxinator start {session}`
5. Gotchas to watch for

## Phase 5 — The ccheck runs automatically

The ccheck (contre-chef) window is part of the tmuxinator config — it starts automatically with `tmuxinator start {session}`. **No manual `/loop` setup needed.** The user does not need to do anything.

The ccheck watches the Chef pane every 30 seconds and:
- **APPROVES** edits in normal zones (src/, tests/, shared-state.md, docs/)
- **SKIPS** edits in sensitive zones (CI, deps, auth, security) — the Chef stays blocked, the user decides when they return
- **LOGS** every decision to `{project}/.claude/ccheck.log` for traceability
- **Re-reads** the sensitive zone list from shared-state.md on every iteration — so the Chef or the user can add zones mid-sprint

The user can leave. The ccheck keeps watch.

**Why this replaced the old `/loop` approach:**
- The `/loop` ran in the user's session, not in tmux → closing the terminal killed it
- The `/loop` was described as optional ("Phase 5") → but without it the Chef blocks
- The `/loop` prompt was frozen at creation → zones added mid-sprint were ignored
- The ccheck is a dedicated window that starts and stops with the tmux session (`tmuxinator start/stop`), reads fresh state every tick, and cannot be accidentally closed

To stop: `CronDelete {job_id}`

## Reference files

| File | Content |
|------|---------|
| `references/gotchas-boss.md` | 16 known pitfalls (G24 — mandatory ccheck, G25 — .claude/ trust guard) |
| `references/templates.md` | Index of all templates |
| `references/shared-state-template.md` | The carnet de cuisine |
| `references/chef-prompt-template.md` | Chef + Sous-Chef + Commis instructions |
| `references/tmuxinator-template.md` | Tmuxinator YAML |
| `references/permissions-template.md` | settings.local.json |
| `references/quality-gates.md` | The tasting card (audit skills) |
| `references/conflict-resolution.md` | Decision tree, coupling matrix, file locking, escalation, stigmergy, patch bankruptcy, divergence, convergence, sprint health |
| `references/anti-patterns-boss.md` | 10 named brigade anti-patterns (Ping-Pong, Ghost Commis, God Commis, etc.) |
| `references/sprint-persistence.md` | Checkpoint, resume, rewind, fresh restart, sprint history (inspired by jj operation log) |
| `references/simplified-model.md` | Stigmergy model for tiers S/M/L — Boids, quorum sensing, DNA repair, reaction-diffusion, apoptosis |
| `references/parallel-exploration.md` | Competing hypotheses, parallel approaches, comparison grid |
| `references/ccheck-prompt-template.md` | Contre-chef prompt (permission auto-approver) |
| `references/raw-tmux-fallback.md` | POSIX bash script equivalent to the tmuxinator YAML (for Ruby-less environments) |

## Integration with other cli-* skills

| Skill | Relation |
|-------|----------|
| `/cli-audit-tangle` | Phase 0 — identify couplings to assign modules to commis without conflicts |
| `/cli-audit-code` | Quality gate per merge — the sous-chef runs it before each merge |
| `/cli-audit-drift` | Quality gate per merge — checks that code honours the contracts |
| `/cli-audit-test` | Quality gate per merge — checks the test pyramid |
| `/cli-audit-sync` | Post-merge quality gate — checks doc/code coherence |
| `/cli-cycle` | End of service — global sprint scorecard |
| `/cli-forge-pipeline` | Optimize the CI pipeline the commis will trigger |
| `/cli-forge-tree` | Validate the project structure before assigning worktrees |
| `/cli-git-conventional` | **ALWAYS** — every commit from the commis and Sous-Chef goes through ghostwriter. Zero AI markers |
| `/cli-forge-doc` | If a quality gate detects missing docs — the Sous-Chef triggers generation |
| `/cli-forge-schema` | If a quality gate detects a missing diagram — Mermaid generation |
