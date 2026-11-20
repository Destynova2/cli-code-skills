<p align="center">
  <strong>cli-code-skills</strong><br>
  <em>Drop-in Claude Code skills that audit your code and generate your docs, designs, and infra.</em>
</p>

<p align="center">
  <a href="LICENSE"><img src="https://img.shields.io/badge/license-MIT-blue.svg" alt="MIT License"></a>
  <img src="https://img.shields.io/badge/skills-38-green.svg" alt="38 Skills">
  <img src="https://img.shields.io/badge/claude--code-skills-8A2BE2" alt="Claude Code Skills">
  <img src="https://img.shields.io/badge/cursor-compatible-F7DF1E" alt="Cursor Compatible">
  <a href="https://github.com/Destynova2/cli-code-skills/stargazers"><img src="https://img.shields.io/github/stars/Destynova2/cli-code-skills?style=flat" alt="Stars"></a>
  <a href="https://github.com/Destynova2/cli-code-skills/network/members"><img src="https://img.shields.io/github/forks/Destynova2/cli-code-skills?style=flat" alt="Forks"></a>
  <a href="https://github.com/Destynova2/cli-code-skills/graphs/contributors"><img src="https://img.shields.io/github/contributors/Destynova2/cli-code-skills" alt="Contributors"></a>
</p>

---

> **`cli-`** = **C**ommand **L**ine **I**nterface + **C**lement **L**iard **I**nitials

## Quickstart

```bash
git clone https://github.com/Destynova2/cli-code-skills.git
cli-code-skills/scripts/install_skills.sh
```

Type `/cli-` in Claude Code or Codex, hit tab, pick a skill:

```
/cli-audit-code src/          → Clean Code score with per-category breakdown
/cli-audit-xray src/core/     → Semantic optimization cards with invariants and validation
/cli-audit-test tests/        → Test plan quality score (ISTQB/TMMi, 12 dimensions)
/cli-forge-hld "payment API"  → HLD with C4 L1-L2, capacity estimation, ADRs
/cli-forge-lld "order service" → LLD with class/sequence diagrams, API specs, DB schemas
/cli-forge-schema "auth flow" → GitHub-compatible Mermaid diagram, auto-split if complex
/cli-cycle .                  → Full project review with prioritized action plan
```

## Skills

### `cli-cycle` — Orchestrator

| Skill | What it does |
|-------|-------------|
| `/cli-cycle [path]` | Runs all applicable cli-* skills, synthesizes a scorecard, and delivers a prioritized action plan. Use with `/loop 7d` for continuous improvement |

### `cli-audit-*` — Analyze & Score

| Skill | What it does |
|-------|-------------|
| `/cli-audit-code [path]` | Scores code against Clean Code principles — 10 categories: naming, functions, DRY, error handling, cognitive load. Works with any language |
| `/cli-audit-data [path]` | Audits PostgreSQL/SQLx invariants, transactions, concurrency, idempotency, migrations, and repair proof |
| `/cli-audit-doc [path]` | Scores documentation quality against RFC 1574, Diataxis, Microsoft M-DOC — 6 categories, any language |
| `/cli-audit-drift [path]` | Detects silent semantic drift between CONTRACTS.md intentions and implementation behavior |
| `/cli-audit-hanoi [path]` | Audits install/config/build step ordering (Dockerfile layers, Ansible tasks, K8s apply order, Rust crates, docs) — stable at the bottom, volatile on top, minimal rework per change |
| `/cli-audit-review [path-or-diff]` | Performs strict MR/PR review with RMI scoring, gate decisions, contract validation, idempotence, proof, CI/release safety, and actionable reviewer comments |
| `/cli-audit-shell [path]` | Audits shell scripts against Google Shell Style Guide and operational safety patterns |
| `/cli-audit-sync [path]` | Verifies doc-code coherence: stale references, broken links, terminology drift, outdated diagrams, non-working examples. 3 layers: structural, semantic, executable |
| `/cli-audit-tangle [path]` | Detects god functions, dependency cycles, dead nodes, and tangled call graph topology |
| `/cli-audit-test [path]` | Scores test plan quality against ISTQB/TMMi — 12 dimensions: techniques (BVA, pairwise, decision table...), pyramid balance, negative testing, NFR, CI integration. Anti-pattern detection |
| `/cli-audit-wizard [path]` | Audits setup/init/doctor/config wizard UX and lifecycle quality |
| `/cli-audit-xray [path]` | Reveals hidden semantic, mathematical, dataflow, and resource-flow structure; produces optimization cards with invariants, risk, and validation |

### `cli-forge-*` — Generate

| Skill | What it does |
|-------|-------------|
| `/cli-forge-arch [system]` | Router — detects HLD vs LLD intent and delegates to the appropriate skill. Use when ambiguous |
| `/cli-forge-chef [sprint]` | Generates a multi-agent Brigade de Cuisine orchestrator with tmux, prompts, worktrees, and gates |
| `/cli-forge-choice-ux [flow]` | Choice UX — simplifies checkout, signup, pricing, settings, configurators, and guided flows using decision-load, friction, accessibility, and conversion heuristics |
| `/cli-forge-demo [project]` | Generates a reproducible live-demo kit, reset path, demo script, and delivery support |
| `/cli-forge-data [change]` | Designs and implements safe PostgreSQL/SQLx schemas, transactions, migrations, and data-state transitions |
| `/cli-forge-doc [repo]` | Generates dual documentation — AI-optimized (AGENTS.md, llms.txt) + human-readable (Diataxis structure) |
| `/cli-forge-github [repo]` | Audits GitHub repository health: rulesets, PR lifecycle, CI checks, releases, branches |
| `/cli-forge-hld [system]` | High-Level Design — C4 L1-L2 diagrams, capacity estimation, ATAM tradeoff analysis, ADRs, deployment architecture. Sources: arc42, Google Design Docs, ByteByteGo |
| `/cli-forge-influence [situation]` | Sales discovery and deal qualification (SPIN, MEDDPICC), feedback, delegation, and influence without authority, when nothing is bargained |
| `/cli-forge-infra [service]` | Ops integration — reads service docs, finds simplest config path, builds dependency trees, proposes upgrades with ADRs |
| `/cli-forge-lld [component]` | Low-Level Design — C4 L3-L4, class/sequence/state diagrams, API contracts (OpenAPI), DB schemas, STRIDE threat model, testability design. Sources: Clean Architecture, DDD, GoF |
| `/cli-forge-nego [situation]` | Negotiation method (Fisher & Ury, Thompson) with Voss as a phrasing toolbox: salary, freelance rate, contracts, scope creep, unpaid invoices, project defense; never invents leverage |
| `/cli-forge-oci-rootless [service]` | Designs migrations from legacy/systemd/bare-metal deployments to rootless OCI operations |
| `/cli-forge-perf [scope]` | Builds performance diagnostics, native benchmark protocols, and optimization validation plans |
| `/cli-forge-pipeline [yaml]` | CI/CD pipeline optimizer using biomimetic patterns (ants, slime mold, bees, mycelium). Works with GitLab CI, GitHub Actions, and any CI system |
| `/cli-forge-plume [message]` | Ghostwrites everyday exchanges — emails, chat, DMs, relances — in the user's voice, zero AI tells, with calibrated humor for follow-ups |
| `/cli-forge-prez [topic]` | Generates technical presentation decks in Marp Markdown |
| `/cli-forge-profile [profile]` | Creates, rewrites, audits, and maintains professional identity content: CV, LinkedIn, GitHub profiles, bios, portfolios, and freelance positioning |
| `/cli-forge-quorum [sprint]` | Generates Byzantine-fault-tolerant REC-Quorum multi-agent orchestration |
| `/cli-forge-readme [path]` | Generates a professional README using the 3-tier pyramid: hook, quickstart, contribute |
| `/cli-forge-resilience [service]` | Resilience & operations — generates prod-parity matrices, runbooks, agent-ready ops packs, failure-injection plans, and incident memory |
| `/cli-forge-schema [desc]` | Generates GitHub-compatible Mermaid diagrams — picks the right type, splits complex ones, converts tables/kanban/PERT + 12 other formats |
| `/cli-forge-tree [path]` | Visualizes, audits, or scaffolds project directory structures with naming conventions |

### Other Skills

| Skill | What it does |
|-------|-------------|
| `/cli-git-conventional` | Enforces Conventional Commits, SemVer, branch naming, and zero AI trailers |
| `/cli-watermark` | Adds and verifies multi-layer code provenance watermarks |

## Naming Convention

```
cli-<action>-<target>
 │    │        │
 │    │        └─ what it operates on (code, doc, hld, lld, infra, readme, schema, tree, test)
 │    └────────── what it does (audit = analyze, forge = generate, cycle = orchestrate)
 └─────────────── namespace
```

## Installation

### Claude Code

```bash
git clone https://github.com/Destynova2/cli-code-skills.git
cli-code-skills/scripts/install_skills.sh --claude
```

### Codex

```bash
git clone https://github.com/Destynova2/cli-code-skills.git
cli-code-skills/scripts/install_skills.sh --codex
```

### Jcode

```bash
cli-code-skills/scripts/install_skills.sh --jcode
```

### Cursor IDE

```bash
git clone https://github.com/Destynova2/cli-code-skills.git
cp -r cli-code-skills/.cursor/rules/* your-project/.cursor/rules/
```

### Update

```bash
cd cli-code-skills && git pull
scripts/install_skills.sh
```

## Validation

Run the repository checks before publishing skill changes:

```bash
python3 scripts/validate_skills.py
```

The validator checks the skill count, frontmatter, README/CLAUDE inventories,
shared-file paths, and Claude/Codex-neutral `cli-cycle` orchestration. The same
check runs in GitHub Actions on pushes and pull requests.

## Project Structure

```
cli-code-skills/
├── .github/workflows/     # Repository validation workflow
├── .claude-plugin/        # Plugin manifest for marketplace
│   └── plugin.json
├── .cursor/rules/         # Cursor IDE rules (4 core skills)
├── docs/reviews/          # cli-cycle audit reports
├── cli-cycle/             # /cli-cycle — orchestrator
├── cli-audit-code/        # /cli-audit-code — Clean Code scoring (CQI)
│   └── references/        #   12 categories, scoring framework, anti-patterns
├── cli-audit-data/        # /cli-audit-data — PostgreSQL/SQLx invariant and concurrency audit
│   └── references/        #   review templates and worked example
├── cli-audit-xray/        # /cli-audit-xray — semantic optimization scanner
│   └── references/        #   SOG, optimization families, examples, risk rules
├── cli-audit-doc/         # /cli-audit-doc — documentation quality (DQI)
│   └── references/        #   12 categories, scoring framework, anti-patterns
├── cli-audit-drift/       # /cli-audit-drift — semantic drift against contracts
│   └── references/        #   contract templates and drift detection rules
├── cli-audit-hanoi/       # /cli-audit-hanoi — ordering & layering audit
│   └── references/        #   canonical stacks and anti-patterns per domain
├── cli-audit-review/      # /cli-audit-review — strict MR/PR review scoring
│   └── references/        #   RMI scoring, conventions, comments, and methodology
├── cli-audit-shell/       # /cli-audit-shell — shell quality and safety
│   └── references/        #   shell categories, scoring, anti-patterns
├── cli-audit-sync/        # /cli-audit-sync — doc-code coherence
│   └── references/        #   coherence layers and duplication rules
├── cli-audit-tangle/      # /cli-audit-tangle — call graph topology
│   └── references/        #   graph methods, analogies, fix patterns
├── cli-audit-test/        # /cli-audit-test — test plan quality & maturity
│   └── references/        #   dimensions, techniques, anti-patterns, scoring
├── cli-audit-wizard/      # /cli-audit-wizard — setup/config UX lifecycle
│   └── references/        #   flows, doctor checks, tooling ladder
├── cli-forge-arch/        # /cli-forge-arch — HLD/LLD router
│   └── references/        #   templates (backward compat)
├── cli-forge-chef/        # /cli-forge-chef — multi-agent brigade orchestrator
│   └── references/        #   prompts, tmuxinator, quality gates
├── cli-forge-choice-ux/   # /cli-forge-choice-ux — choice, checkout, signup, settings UX
│   └── references/        #   principles, scorecard, flow patterns
├── cli-forge-demo/        # /cli-forge-demo — reproducible live-demo kit
│   └── references/        #   staging, dramaturgy, tooling, templates
├── cli-forge-data/        # /cli-forge-data — safe PostgreSQL/SQLx design and implementation
├── cli-forge-github/      # /cli-forge-github — GitHub repository health
│   └── references/        #   repo health patterns and checks
├── cli-forge-hld/         # /cli-forge-hld — High-Level Design
│   └── references/        #   sections, scoring, anti-patterns, estimation
├── cli-forge-doc/         # /cli-forge-doc — full project documentation
│   └── references/        #   Diataxis framework templates
├── cli-forge-influence/   # /cli-forge-influence — sales discovery, feedback, delegation
│   └── references/        #   b2b sales, management, sources
├── cli-forge-infra/       # /cli-forge-infra — ops integration
│   └── references/        #   scoring, patterns, checklists
├── cli-forge-lld/         # /cli-forge-lld — Low-Level Design
│   └── references/        #   sections, scoring, anti-patterns, LLD template
├── cli-forge-nego/        # /cli-forge-nego — negotiation prep, replies, role-play
│   └── references/        #   prep, techniques, anti-patterns, project defense
├── cli-forge-oci-rootless/ # /cli-forge-oci-rootless — rootless OCI migration
│   └── references/        #   scenarios, proof gates, handoffs
├── cli-forge-perf/        # /cli-forge-perf — performance optimization method
│   └── references/        #   profiling, benchmarks, domains, examples
├── cli-forge-pipeline/    # /cli-forge-pipeline — CI/CD pipeline optimizer (biomimetic)
│   └── references/        #   pipeline templates, scoring, cache strategy
├── cli-forge-plume/       # /cli-forge-plume — ghostwritten exchanges and relances
│   └── references/        #   AI-tells catalog, relance ladder, humor bank
├── cli-forge-prez/        # /cli-forge-prez — technical slide decks
│   └── references/        #   slide structure and presentation patterns
├── cli-forge-profile/     # /cli-forge-profile — professional identity profiles
│   └── references/        #   platform, editorial, and method guidance
├── cli-forge-quorum/      # /cli-forge-quorum — REC-Quorum orchestration
│   └── references/        #   quorum phases, templates, gotchas
├── cli-forge-readme/      # /cli-forge-readme — README generation (RCI)
│   └── references/        #   README pyramid and project types
├── cli-forge-resilience/  # /cli-forge-resilience — resilience, runbooks, agent-ready ops
│   └── references/        #   models, tests, runbook templates, agent ops
├── cli-forge-schema/      # /cli-forge-schema — Mermaid diagrams
│   └── references/        #   diagram types, conversions, modes
├── cli-forge-tree/        # /cli-forge-tree — directory structure (SHS)
│   └── references/        #   archetypes, conventions
├── cli-git-conventional/  # /cli-git-conventional — commits, SemVer, branches
│   └── references/        #   branch, command, and SemVer rules
├── cli-watermark/         # /cli-watermark — code provenance watermarking
│   └── references/        #   watermark layers and proof bundle
├── shared/                # Shared cross-skill contracts
├── scripts/               # Installation and repository validation utilities
├── gotchas.md             # Persistent lessons-learned, read by all skills
└── README.md
```

Each skill is a self-contained directory:

```
cli-<skill>/
├── SKILL.md          # Prompt: workflow, checklist, principles
└── references/       # Domain knowledge: templates, standards, cheatsheets
```

Skills run as forked agents with scoped tool access. They read your code, produce structured markdown with scores, diagrams, and actionable next steps.

## Contributing

PRs welcome. To add a skill:

1. Create a directory following `cli-<action>-<target>`
2. Add a `SKILL.md` with frontmatter (`name`, `description`, `context: fork`)
3. Optionally add `references/` for domain knowledge
4. Submit a PR

## License

MIT
