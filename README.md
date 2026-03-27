<p align="center">
  <strong>cli-code-skills</strong><br>
  <em>Drop-in Claude Code skills that audit your code and generate your docs, designs, and infra.</em>
</p>

<p align="center">
  <a href="LICENSE"><img src="https://img.shields.io/badge/license-MIT-blue.svg" alt="MIT License"></a>
  <img src="https://img.shields.io/badge/skills-14-green.svg" alt="14 Skills">
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
cp -r cli-code-skills/cli-* ~/.claude/skills/
```

Type `/cli-` in Claude Code, hit tab, pick a skill:

```
/cli-audit-code src/          → Clean Code score with per-category breakdown
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
| `/cli-audit-doc [path]` | Scores documentation quality against RFC 1574, Diataxis, Microsoft M-DOC — 6 categories, any language |
| `/cli-audit-sync [path]` | Verifies doc-code coherence: stale references, broken links, terminology drift, outdated diagrams, non-working examples. 3 layers: structural, semantic, executable |
| `/cli-audit-test [path]` | Scores test plan quality against ISTQB/TMMi — 12 dimensions: techniques (BVA, pairwise, decision table...), pyramid balance, negative testing, NFR, CI integration. Anti-pattern detection |

### `cli-forge-*` — Generate

| Skill | What it does |
|-------|-------------|
| `/cli-forge-hld [system]` | High-Level Design — C4 L1-L2 diagrams, capacity estimation, ATAM tradeoff analysis, ADRs, deployment architecture. Sources: arc42, Google Design Docs, ByteByteGo |
| `/cli-forge-lld [component]` | Low-Level Design — C4 L3-L4, class/sequence/state diagrams, API contracts (OpenAPI), DB schemas, STRIDE threat model, testability design. Sources: Clean Architecture, DDD, GoF |
| `/cli-forge-arch [system]` | Router — detects HLD vs LLD intent and delegates to the appropriate skill. Use when ambiguous |
| `/cli-forge-doc [repo]` | Generates dual documentation — AI-optimized (AGENTS.md, llms.txt) + human-readable (Diataxis structure) |
| `/cli-forge-infra [service]` | Ops integration — reads service docs, finds simplest config path, builds dependency trees, proposes upgrades with ADRs |
| `/cli-forge-readme [path]` | Generates a professional README using the 3-tier pyramid: hook, quickstart, contribute |
| `/cli-forge-schema [desc]` | Generates GitHub-compatible Mermaid diagrams — picks the right type, splits complex ones, converts tables/kanban/PERT + 12 other formats |
| `/cli-forge-pipeline [yaml]` | CI/CD pipeline optimizer using biomimetic patterns (ants, slime mold, bees, mycelium). Works with GitLab CI, GitHub Actions, and any CI system |
| `/cli-forge-tree [path]` | Visualizes, audits, or scaffolds project directory structures with naming conventions |

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
cp -r cli-code-skills/cli-* ~/.claude/skills/
```

### Cursor IDE

```bash
git clone https://github.com/Destynova2/cli-code-skills.git
cp -r cli-code-skills/.cursor/rules/* your-project/.cursor/rules/
```

### Update

```bash
cd cli-code-skills && git pull
cp -r cli-* ~/.claude/skills/
```

## Project Structure

```
cli-code-skills/
├── .claude-plugin/        # Plugin manifest for marketplace
│   └── plugin.json
├── .cursor/rules/         # Cursor IDE rules (4 core skills)
├── cli-cycle/             # /cli-cycle — orchestrator
├── cli-audit-code/        # /cli-audit-code — Clean Code scoring (CQI)
│   └── references/        #   12 categories, scoring framework, anti-patterns
├── cli-audit-doc/         # /cli-audit-doc — documentation quality (DQI)
│   └── references/        #   12 categories, scoring framework, anti-patterns
├── cli-audit-sync/        # /cli-audit-sync — doc-code coherence
├── cli-audit-test/        # /cli-audit-test — test plan quality & maturity
│   └── references/        #   dimensions, techniques, anti-patterns, scoring
├── cli-forge-arch/        # /cli-forge-arch — HLD/LLD router
│   └── references/        #   templates (backward compat)
├── cli-forge-hld/         # /cli-forge-hld — High-Level Design
│   └── references/        #   sections, scoring, anti-patterns, estimation
├── cli-forge-lld/         # /cli-forge-lld — Low-Level Design
│   └── references/        #   sections, scoring, anti-patterns, LLD template
├── cli-forge-doc/         # /cli-forge-doc — full project documentation
│   └── references/        #   Diataxis framework templates
├── cli-forge-infra/       # /cli-forge-infra — ops integration
│   └── references/        #   scoring, patterns, checklists
├── cli-forge-pipeline/    # /cli-forge-pipeline — CI/CD pipeline optimizer (biomimetic)
├── cli-forge-readme/      # /cli-forge-readme — README generation (RCI)
├── cli-forge-schema/      # /cli-forge-schema — Mermaid diagrams
│   └── references/        #   diagram types, conversions, modes
├── cli-forge-tree/        # /cli-forge-tree — directory structure (SHS)
│   └── references/        #   archetypes, conventions
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
