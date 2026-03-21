<p align="center">
  <strong>cli-code-skills</strong><br>
  <em>Drop-in Claude Code skills that audit your code and generate your docs, designs, and infra.</em>
</p>

<p align="center">
  <a href="LICENSE"><img src="https://img.shields.io/badge/license-MIT-blue.svg" alt="MIT License"></a>
  <img src="https://img.shields.io/badge/skills-9-green.svg" alt="9 Skills">
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
/cli-forge-arch "payment API" → Full HLD with C4 diagrams and capacity estimations
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

### `cli-forge-*` — Generate

| Skill | What it does |
|-------|-------------|
| `/cli-forge-arch [system]` | Generates HLD/LLD design docs with C4 diagrams, ADRs, and back-of-envelope capacity estimations (ATAM framework) |
| `/cli-forge-doc [repo]` | Generates dual documentation — AI-optimized (AGENTS.md, llms.txt) + human-readable (Diataxis structure) |
| `/cli-forge-infra [service]` | Ops integration — reads service docs, finds simplest config path, builds dependency trees, proposes upgrades with ADRs |
| `/cli-forge-readme [path]` | Generates a professional README using the 3-tier pyramid: hook, quickstart, contribute |
| `/cli-forge-schema [desc]` | Generates GitHub-compatible Mermaid diagrams — picks the right type, splits complex ones, converts tables/kanban/PERT + 12 other formats |
| `/cli-forge-tree [path]` | Visualizes, audits, or scaffolds project directory structures with naming conventions |

## Naming Convention

```
cli-<action>-<target>
 │    │        │
 │    │        └─ what it operates on (code, doc, arch, infra, readme, schema, tree)
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
├── cli-audit-code/        # /cli-audit-code — Clean Code scoring
├── cli-audit-doc/         # /cli-audit-doc — documentation quality
├── cli-forge-arch/        # /cli-forge-arch — HLD/LLD/ADR generation
│   └── references/        #   templates: HLD, LLD, estimation cheatsheet
├── cli-forge-doc/         # /cli-forge-doc — full project documentation
├── cli-forge-infra/       # /cli-forge-infra — ops integration
├── cli-forge-readme/      # /cli-forge-readme — README generation
├── cli-forge-schema/      # /cli-forge-schema — Mermaid diagrams
│   └── references/        #   diagram types, palettes, GitHub limits
├── cli-forge-tree/        # /cli-forge-tree — directory structure
└── README.md
```

Each skill is a self-contained directory:

```
cli-<skill>/
├── SKILL.md          # Prompt: workflow, checklist, principles
└── reference.md      # Domain knowledge: templates, standards, cheatsheets
```

Skills run as forked agents with scoped tool access. They read your code, produce structured markdown with scores, diagrams, and actionable next steps.

## Contributing

PRs welcome. To add a skill:

1. Create a directory following `cli-<action>-<target>`
2. Add a `SKILL.md` with frontmatter (`name`, `description`, `context: fork`)
3. Optionally add `reference.md` for domain knowledge
4. Submit a PR

## License

MIT
