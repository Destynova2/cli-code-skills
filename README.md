<p align="center">
  <strong>cli-code-skills</strong><br>
  <em>Production-ready Claude Code skills — audit, forge, ship.</em>
</p>

<p align="center">
  <a href="LICENSE"><img src="https://img.shields.io/badge/license-MIT-blue.svg" alt="MIT License"></a>
  <img src="https://img.shields.io/badge/skills-5-green.svg" alt="5 Skills">
  <img src="https://img.shields.io/badge/claude--code-skills-8A2BE2" alt="Claude Code">
</p>

---

> **`cli-`** stands for **C**ommand **L**ine **I**nterface — and **C**lement **L**iard's **I**nitials.

## Quick Start

```bash
git clone https://github.com/Destynova2/cli-code-skills.git
cp -r cli-code-skills/cli-* ~/.claude/skills/
```

Type `/cli-` in Claude Code and tab-complete.

## Skills

### 🔍 `cli-audit-*` — Analysis

| Skill | What it does |
|-------|-------------|
| `/cli-audit-code [path]` | Scores code against Clean Code principles (10 categories: naming, functions, DRY, error handling, cognitive load...) |
| `/cli-audit-doc [path]` | Scores documentation quality against RFC 1574, Diataxis, Microsoft M-DOC (6 categories, any language) |

### 🔨 `cli-forge-*` — Generation

| Skill | What it does |
|-------|-------------|
| `/cli-forge-arch [system]` | Generates HLD/LLD design docs with C4 diagrams, ADRs, and back-of-envelope capacity estimations |
| `/cli-forge-doc [repo]` | Generates dual documentation — AI-optimized (AGENTS.md, llms.txt) + human-readable (Diataxis) |
| `/cli-forge-infra [service]` | Ops integration — finds simplest config path, builds dependency trees, proposes upgrades with ADRs |

## Naming Convention

```
cli-<action>-<target>
 │    │        │
 │    │        └─ what it operates on (code, doc, arch, infra)
 │    └────────── what it does (audit = analyze, forge = generate)
 └─────────────── namespace (CLI + Clement Liard Initials)
```

## Installation

Copy individual skills or all at once:

```bash
# All skills
cp -r cli-code-skills/cli-* ~/.claude/skills/

# Just what you need
cp -r cli-code-skills/cli-audit-code ~/.claude/skills/
cp -r cli-code-skills/cli-forge-arch ~/.claude/skills/
```

## How It Works

Each skill is a directory with:

```
cli-<skill>/
├── SKILL.md          # Main prompt (workflow, checklist, principles)
└── reference.md      # Domain knowledge (templates, standards, cheatsheets)
```

Skills run as forked agents with scoped tool access — they read your code, produce structured markdown reports with scores, diagrams, and actionable next steps.

## Contributing

PRs welcome. Follow the naming convention `cli-<action>-<target>`.

## License

MIT
