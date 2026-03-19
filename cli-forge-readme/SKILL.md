---
name: cli-forge-readme
description: "Use this skill whenever the user wants to create, improve, audit, or rewrite a README.md file for any project. Triggers include: 'readme', 'README', 'documentation for my project', 'write a readme', 'improve my readme', 'project landing page', or any request to document a codebase, library, CLI tool, infrastructure project, or research repo. Also triggers when the user asks to 'make my project more accessible', 'add a getting started guide', or wants badges, installation instructions, or contributing guidelines. Use this skill even when the user just says 'document this' or 'make this repo presentable'. Do NOT use for API reference docs generation, full documentation sites (mdbook, docusaurus), or man pages."
---

> **Language rule:** Detect the project's primary language (from README, comments, docs, commit messages). Output the README in that language. If the project is bilingual, ask the user which language to use before proceeding.

# README Generator — Production First

Generate professional README.md files where **results come first, plumbing comes second**.

## Philosophy

A README is a **landing page**, not a technical manual.
- 90% of visitors want to know: what, why, how to start
- 10% want to contribute or understand internals
- Structure for the 90%, don't punish them with the 10%

**Tone rule: Friendly on the surface, technical in depth.**
- Tier 1–2: plain language, no jargon, a junior dev or a PM can understand
- Tier 3: full technical depth, assume the reader codes

---

## Workflow

### Step 1 — Detect project context

Read the project root to auto-detect the type:

| Signal | Project Type |
|--------|-------------|
| `Cargo.toml` | `rust-lib` or `rust-cli` (check `[[bin]]`) |
| `go.mod` | `go-lib` or `go-cli` |
| `package.json` | `js-lib`, `js-app`, or `js-cli` (check `bin` field) |
| `pyproject.toml` / `setup.py` | `python-lib` or `python-cli` |
| `*.tf` / `terragrunt.hcl` | `infra-terraform` |
| `helmfile.yaml` / `Chart.yaml` | `infra-helm` |
| `docker-compose*.yml` + app code | `app-backend` |
| `Containerfile` / `Dockerfile` | `container` (prefer Containerfile naming) |
| `*.container` / `*.kube` | `podman-quadlet` |
| `kustomization.yaml` | `kustomize` (+ `flux` if `flux-system/` present) |
| `site.yml` / `playbooks/` / `roles/` | `ansible` |
| `*.sh` + no other code | `shell-scripts` |
| `flake.nix` / `shell.nix` | add `nix` flag |
| `.github/workflows/` | add `ci-github` flag |
| `.woodpecker/` / `.forgejo/` | add `ci-forgejo` flag |

**If nothing detected or directory is empty** → use `ask_user_input` tool with QCM:

```
Q1: "What type of project?" → [Library/SDK, CLI tool, API/Backend, Infrastructure/IaC, Monorepo, Other]
Q2: "Primary language?" → [Rust, Go, Python, TypeScript/JS, Shell/Bash, HCL/Terraform, Other]
Q3: "Is this deployed in production?" → [Yes already, Not yet but planned, No it's a lib/tool]
```

### Step 2 — Gather content

Before writing anything:

1. **Read existing files**: README.md (current), CHANGELOG, LICENSE, CI configs, manifest files (Cargo.toml / package.json / pyproject.toml) for name, description, version
2. **Only ask for what's missing**: don't re-ask what the codebase already tells you
3. **Find the hero moment**: the ONE thing that makes someone go "I need this"

### Step 3 — Write using the 3-Tier Pyramid

---

## The 3-Tier Pyramid

### TIER 1 — The Hook (everyone reads this)

```markdown
# Project Name

> One-liner: what it does, for whom, in plain language.

[badges: CI status | version | license — max 4]

[Optional: 1 screenshot, GIF, or architecture diagram]
```

**Rules:**
- Title = project name, nothing else
- One-liner: zero jargon. A PM or a junior can understand it
- Badges: only actionable ones. No vanity badges
- Visual: terminal GIF (CLIs), diagram (infra/libs), screenshot (apps)

---

### TIER 2 — Get Started (most people stop here)

```markdown
## Quickstart

[3 commands max → visible result]

## What it does

[Feature list — short, scannable, max 5 items]

## Examples

[Real-world usage with expected output shown]
```

**Rules:**
- Quickstart must show a **visible result** — not just "it compiled"
- Show actual output: terminal for CLIs, curl response for APIs, `terraform output` for IaC
- Max 5 features listed. Link to docs for the rest
- Examples use realistic data, not `foo/bar/baz`

---

### TIER 3 — Contribute & Understand (devs / QA only)

```markdown
## Project Structure

[Simplified tree, max 2 levels, every folder annotated in ≤5 words]

## Development

### Prerequisites
### Build & Run  
### Tests
### Linting / QA

## Architecture

[Only if non-obvious. Link to ADRs if they exist]

## Contributing

## Roadmap

## License
```

**Rules:**
- Project Structure: annotated `tree`, not a wall of paths
- Dev commands: copy-paste ready. Show what a passing test run looks like
- No dead sections: if you can't fill it, don't include it
- Contributing: link to CONTRIBUTING.md or 3–5 lines inline

---

## Project Type Adaptations

### Libraries (`rust-lib` / `go-lib` / `python-lib` / `js-lib`)
- **Tier 2 Quickstart**: package manager install + minimal code example with output
- **Tier 2 extra**: "API Overview" — top 3–5 public types/functions
- **Tier 3 extra**: Minimum Supported Version (MSRV / Go / Python)
- **Badge**: crates.io / pkg.go.dev / PyPI / npm

### CLI tools (`rust-cli` / `go-cli` / `python-cli`)
- **Tier 1**: terminal GIF or asciinema strongly recommended
- **Tier 2 Quickstart**: install method + one command with output
- **Tier 2 extra**: "Usage" with flags/subcommands table, "Configuration" if config file exists

### Infrastructure (`infra-terraform` / `infra-helm`)
- **Tier 1**: architecture diagram (mermaid or ASCII)
- **Tier 2 Quickstart**: apply/install + how to verify it works
- **Tier 2 extra**: Inputs/Outputs table (TF) or Values table (Helm), Requirements
- **Tier 3 extra**: State Management, Secrets Handling

### Backend apps (`app-backend`)
- **Tier 2 Quickstart**: `docker compose up` → curl that proves it works
- **Tier 2 extra**: API Endpoints summary table, Environment Variables table
- **Tier 3 extra**: Deployment, Monitoring, Database Migrations

### Containers (`container` / `podman-quadlet`)
- **Tier 2 Quickstart**: `podman run` / `docker run` with all required flags shown
- **Tier 2 extra**: Environment Variables, Volumes, Ports tables
- **Tier 2 extra for Quadlet**: show `systemctl --user start app.service` flow

### Kustomize / Flux (`kustomize` / `flux`)
- **Tier 1**: architecture diagram (what gets deployed where)
- **Tier 2 Quickstart**: `kubectl apply -k overlays/dev/` or `flux reconcile`
- **Tier 2 extra**: Overlays table (env → what changes), Prerequisites (cluster, Flux bootstrap)
- **Tier 3 extra**: "GitOps Workflow" (push → reconcile → verify), "Secrets Management"

### Ansible (`ansible`)
- **Tier 1**: what the playbooks configure + target infrastructure
- **Tier 2 Quickstart**: `ansible-playbook -i inventory/dev playbooks/site.yml`
- **Tier 2 extra**: "Roles" table (role → what it does), "Inventory Structure"
- **Tier 2 extra**: "Testing" if Molecule present (`molecule test`)
- **Tier 3 extra**: "Variables Precedence", "Vault Secrets", "CI Integration"

### Shell scripts (`shell-scripts`)
- **Tier 2 Quickstart**: `./bin/setup.sh` with expected output
- **Tier 2 extra**: "Scripts" table (script → purpose → args)
- **Tier 3 extra**: "Dependencies" (required tools), "Testing" (bats/shellspec)

### Monorepos
- **Tier 1**: overview of what's in the repo
- **Tier 2**: table of packages/services, one-liner each, links to sub-READMEs
- **Tier 3**: Workspace Setup, Inter-package Dependencies

---

## Formatting Standards

1. `#` = project name only. `##` = sections. `###` = subsections. Never deeper than `####`
2. Code blocks: always specify language (`bash`, `rust`, `yaml`...)
3. Tables: for structured data only (env vars, flags, endpoints). Never for prose
4. Links: relative for in-repo, absolute for external
5. Target length: Tier 1+2 fit in ~3 screen heights. Tier 3 can expand
6. Language: match project's audience. Default English for open-source
7. No empty sections: if nothing to say, omit entirely. No TODOs in shipped READMEs

## Useful Badges (copy-paste)

```markdown
[![CI](https://github.com/USER/REPO/actions/workflows/ci.yml/badge.svg)](https://github.com/USER/REPO/actions)
[![Crates.io](https://img.shields.io/crates/v/NAME)](https://crates.io/crates/NAME)
[![npm](https://img.shields.io/npm/v/NAME)](https://www.npmjs.com/package/NAME)
[![PyPI](https://img.shields.io/pypi/v/NAME)](https://pypi.org/project/NAME)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![License: AGPL-3.0](https://img.shields.io/badge/License-AGPL--3.0-blue.svg)](LICENSE)
[![MSRV](https://img.shields.io/badge/MSRV-1.XX-blue)](Cargo.toml)
[![Docs](https://docs.rs/NAME/badge.svg)](https://docs.rs/NAME)
```

---

## Output

- Write to project root as `README.md`, or to `/mnt/user-data/outputs/README.md` if no project
- Show a brief summary: which tier sections were generated, what's missing/skipped
- If the `project-tree` skill was used, embed the annotated tree in Tier 3 "Project Structure"
