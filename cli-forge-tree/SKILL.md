---
name: cli-forge-tree
description: "Use this skill whenever the user wants to visualize, generate, audit, or scaffold a project directory structure. Triggers include: 'project structure', 'folder structure', 'directory layout', 'tree', 'arborescence', 'scaffold', 'init project', 'organize my files', 'naming conventions', or any request to understand or create how files and folders should be organized in a codebase. Also triggers when someone says 'create a new project', 'bootstrap', 'init', or asks 'where should I put this file'. Use for any language or framework. Do NOT use for file system operations unrelated to project organization (like disk cleanup or backup scripts)."
---

> **Language rule:** Detect the project's primary language (from README, comments, docs, commit messages). Output your report in that language. If the project is bilingual, ask the user which language to use before proceeding.

# Project Tree — Structure & Scaffold

Visualize, audit, or scaffold project directory structures with **human-readable naming** that any skill level can navigate.

## Core Naming Conventions

These rules apply universally, regardless of language or framework.

### 1. Case & Separators

| Rule | Example | Why |
|------|---------|-----|
| **lowercase-kebab-case** for folders | `api-gateway/` | No OS ambiguity, readable |
| **lowercase-kebab-case** for files | `user-service.ts` | Consistent with folders |
| **UPPER_CASE** only for root meta-files | `README.md`, `LICENSE`, `CHANGELOG.md` | Convention, instantly visible |
| **dot-prefix** for config/hidden files | `.env`, `.gitignore`, `.editorconfig` | Standard convention |

**Exceptions** (follow ecosystem conventions when they're universal):
- `Cargo.toml`, `Makefile`, `Dockerfile`, `Jenkinsfile` — capitalized by convention
- `package.json`, `tsconfig.json` — ecosystem standard
- `__init__.py`, `__main__.py` — Python convention
- `lib.rs`, `main.rs`, `mod.rs` — Rust convention

### 2. Naming Principles

| Principle | Good | Bad | Why |
|-----------|------|-----|-----|
| **Descriptive but short** | `docs/` | `documentation-files/` | 3-8 chars ideal for top-level |
| **Role-obvious** | `scripts/` | `scr/` | No abbreviation puzzles |
| **No redundancy** | `src/auth/` | `src/auth-module/` | Parent gives context |
| **No version in names** | `api/` | `api-v2/` | Use git branches/tags |
| **Plural for collections** | `tests/`, `scripts/` | `test/`, `script/` | Unless ecosystem convention says otherwise |

### 3. Depth Rule

**Max 3 levels before you need a good reason.**

```
✅ src/api/handlers/          ← 3 levels, fine
❌ src/api/handlers/v2/users/ ← 5 levels, rethink your structure
```

If you need 4+ levels, consider: is this a separate package/crate/module?

---

## Workflow

### Step 1 — Detect or Ask

**If a project exists** → read the root directory and detect:

| Signal | Project Archetype |
|--------|------------------|
| `Cargo.toml` | `rust` |
| `go.mod` | `go` |
| `package.json` | `js/ts` (check for framework: next, express, etc.) |
| `pyproject.toml` / `setup.py` | `python` |
| `*.tf` | `terraform` |
| `Chart.yaml` / `helmfile.yaml` | `helm` |
| `kustomization.yaml` | `kustomize` (check for `flux-system/` → `flux`) |
| `Containerfile` / `Dockerfile` | `oci-container` (prefer `Containerfile` naming for Podman/OCI) |
| `*.container` / `*.kube` | `podman-quadlet` |
| `site.yml` / `playbooks/` / `roles/` | `ansible` (check for `molecule/` → add `molecule` flag) |
| `docker-compose*.yml` | `docker-compose` |
| `*.sh` + no other code | `shell-scripts` |
| `flake.nix` | `nix` |
| Multiple of the above | `monorepo` |

Then: display the current tree (annotated), highlight issues, suggest improvements.

**If no project or empty directory** → use `ask_user_input` QCM:

```
Q1: "What are you building?" → [Library/SDK, CLI tool, API/Backend service, Infrastructure (Terraform/K8s), Full-stack app, Monorepo, Other]
Q2: "Primary language/stack?" → [Rust, Go, Python, TypeScript/Node, Bash/Shell, Terraform/HCL, Other]
Q3: "Extras to include?" (multi-select) → [Docker, CI/CD configs, Documentation folder, Nix/devshell, Benchmarks, Examples]
```

### Step 2 — Generate or Audit

**Generate mode** (new project): create the tree structure using the archetype templates below, then scaffold using `mkdir -p` and touch for placeholder files.

**Audit mode** (existing project): display annotated tree, then list:
- Naming violations (wrong case, unclear names, too deep)
- Missing standard files (README, LICENSE, .gitignore)
- Suggested reorganization if structure is unclear

### Step 3 — Display

Always show the tree in this annotated format:

```
project-name/
├── src/                    # Source code
│   ├── lib.rs              # Library entry point
│   └── handlers/           # Request handlers
├── tests/                  # Integration tests
├── docs/                   # Documentation
├── scripts/                # Dev & CI scripts
├── .github/                # GitHub Actions CI
│   └── workflows/
├── Cargo.toml              # Manifest & deps
├── README.md               # Project landing page
├── LICENSE                 # AGPL-3.0
└── .gitignore              # Git exclusions
```

**Rules for annotations:**
- Every folder gets a `# comment` — 3-6 words max
- Files only get comments if their role isn't obvious from the name
- Use the exact `├──` / `└──` / `│` box-drawing characters
- Max 2 levels shown by default. Deeper = collapsed with `...`

---

## Archetype Templates

### `rust` (lib or CLI)

```
project-name/
├── src/
│   ├── lib.rs              # Public API
│   ├── main.rs             # CLI entry (if bin)
│   └── config.rs           # Configuration
├── tests/                  # Integration tests
├── benches/                # Benchmarks (optional)
├── examples/               # Usage examples
├── docs/                   # Additional docs
├── scripts/                # Dev helpers
├── Cargo.toml
├── README.md
├── LICENSE
├── CHANGELOG.md
└── .gitignore
```

### `go` (lib or CLI)

```
project-name/
├── cmd/                    # CLI entry points
│   └── project-name/
│       └── main.go
├── internal/               # Private packages
├── pkg/                    # Public packages (if lib)
├── api/                    # API definitions (proto, openapi)
├── configs/                # Config templates
├── scripts/                # Dev helpers
├── docs/                   # Documentation
├── go.mod
├── go.sum
├── README.md
├── LICENSE
└── .gitignore
```

### `python` (lib or CLI)

```
project-name/
├── src/
│   └── project_name/       # Note: underscore for Python packages
│       ├── __init__.py
│       ├── __main__.py      # CLI entry (if CLI)
│       └── core.py
├── tests/
├── docs/
├── scripts/
├── pyproject.toml           # PEP 621 manifest
├── README.md
├── LICENSE
└── .gitignore
```

### `js-ts` (lib, CLI, or app)

```
project-name/
├── src/
│   ├── index.ts             # Entry point
│   ├── cli.ts               # CLI entry (if CLI)
│   └── utils/
├── tests/
├── docs/
├── scripts/
├── package.json
├── tsconfig.json
├── README.md
├── LICENSE
└── .gitignore
```

### `terraform`

```
infra-project/
├── modules/                 # Reusable modules
│   ├── network/
│   └── compute/
├── envs/                    # Per-environment configs
│   ├── dev/
│   ├── staging/
│   └── prod/
├── scripts/                 # Helper scripts
├── docs/                    # Arch docs, ADRs
│   └── adr/
├── main.tf                  # Root module
├── variables.tf             # Input variables
├── outputs.tf               # Outputs
├── versions.tf              # Provider versions
├── backend.tf               # State backend
├── README.md
└── .gitignore
```

### `helm`

```
chart-name/
├── charts/                  # Sub-chart dependencies
├── templates/               # K8s manifests
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   └── _helpers.tpl
├── values.yaml              # Default values
├── values-prod.yaml         # Prod overrides
├── Chart.yaml               # Chart metadata
├── README.md
└── .helmignore
```

### `app-backend` (Docker Compose based)

```
project-name/
├── src/                     # Application code
│   └── ...
├── migrations/              # DB migrations
├── scripts/                 # Dev & ops scripts
├── configs/                 # App config templates
├── docs/                    # Documentation
├── docker/                  # Dockerfiles per service
│   ├── app.dockerfile
│   └── worker.dockerfile
├── docker-compose.yml       # Dev environment
├── docker-compose.prod.yml  # Prod overrides
├── README.md
├── LICENSE
└── .gitignore
```

### `monorepo`

```
project-name/
├── apps/                    # Deployable applications
│   ├── api/
│   └── web/
├── libs/                    # Shared libraries
│   ├── core/
│   └── utils/
├── infra/                   # Infrastructure code
├── scripts/                 # Repo-wide scripts
├── docs/                    # Global docs
├── .github/                 # CI/CD
│   └── workflows/
├── README.md                # Root overview + links to sub-READMEs
├── LICENSE
└── .gitignore
```

### `oci-container` (Podman / Docker / OCI)

Use `Containerfile` (OCI standard) over `Dockerfile` when targeting Podman or multi-runtime.

```
project-name/
├── src/                     # Application code
│   └── ...
├── build/                   # Build contexts per target
│   ├── app.containerfile    # Main app image
│   └── worker.containerfile # Worker image (if any)
├── configs/                 # Runtime config templates
├── scripts/                 # Build & push helpers
├── tests/                   # Integration tests
├── compose.yml              # Dev environment (Podman Compose / Docker Compose)
├── compose.prod.yml         # Prod overrides
├── README.md
├── LICENSE
└── .gitignore
```

**Naming notes:**
- `Containerfile` > `Dockerfile` for OCI portability (Podman auto-detects both)
- `compose.yml` > `docker-compose.yml` (Docker Compose V2+ standard)
- Use `.containerfile` extension when multiple build targets exist

### `kustomize` / `flux` (GitOps Kubernetes)

```
infra-project/
├── base/                    # Shared manifests
│   ├── deployment.yaml
│   ├── service.yaml
│   └── kustomization.yaml
├── overlays/                # Per-environment patches
│   ├── dev/
│   │   ├── kustomization.yaml
│   │   └── patch-replicas.yaml
│   ├── staging/
│   └── prod/
├── components/              # Reusable Kustomize components
├── flux-system/             # Flux bootstrap (if Flux)
│   ├── gotk-components.yaml
│   └── gotk-sync.yaml
├── clusters/                # Cluster-specific Flux configs (if multi-cluster)
│   ├── dev/
│   └── prod/
├── docs/                    # Architecture, ADRs
├── scripts/                 # Validation, diff helpers
├── README.md
└── .gitignore
```

### `ansible` (with optional Molecule)

```
ansible-project/
├── playbooks/               # Top-level playbooks
│   ├── site.yml
│   ├── deploy.yml
│   └── rollback.yml
├── roles/                   # Ansible roles
│   └── my-role/
│       ├── tasks/
│       │   └── main.yml
│       ├── handlers/
│       ├── templates/
│       ├── files/
│       ├── vars/
│       ├── defaults/
│       ├── meta/
│       └── molecule/        # Molecule test scenarios
│           └── default/
│               ├── molecule.yml
│               ├── converge.yml
│               └── verify.yml
├── inventory/               # Host inventories
│   ├── dev/
│   │   ├── hosts.yml
│   │   └── group_vars/
│   ├── staging/
│   └── prod/
├── collections/             # requirements.yml for Galaxy
├── scripts/                 # Wrapper scripts
├── docs/                    # Runbook, architecture
├── ansible.cfg              # Project-level Ansible config
├── requirements.yml         # Galaxy dependencies
├── README.md
└── .gitignore
```

**Naming notes:**
- `hosts.yml` > `hosts.ini` (YAML inventories are more maintainable)
- Role names: `lowercase-kebab-case` or `lowercase_snake_case` (Ansible convention allows both, pick one)
- Molecule scenarios go inside their role, not at project root

### `podman-quadlet` (systemd integration)

For Podman containers managed via systemd Quadlet units.

```
project-name/
├── containers/              # Quadlet unit files
│   ├── app.container        # Podman container unit
│   ├── app.kube             # Podman kube play unit
│   └── app.volume           # Podman volume unit
├── kube/                    # K8s YAML for kube play
│   └── app-pod.yml
├── configs/                 # App configuration
├── scripts/                 # Install / deploy helpers
├── docs/                    # Architecture
├── README.md
└── .gitignore
```

### `shell-scripts` (Bash / POSIX sh)

```
project-name/
├── bin/                     # Main executable scripts
│   ├── setup.sh
│   └── deploy.sh
├── lib/                     # Shared functions (sourced)
│   ├── logging.sh
│   └── utils.sh
├── tests/                   # Test scripts (bats or shellspec)
├── docs/                    # Usage guides
├── completions/             # Shell completions (bash, zsh, fish)
├── README.md
├── LICENSE
└── .gitignore
```

**Naming notes:**
- Always `.sh` extension (even for bash — makes linting with shellcheck easier)
- `bin/` for executables, `lib/` for sourced helpers
- All scripts start with `#!/usr/bin/env bash` (or `#!/bin/sh` for POSIX)
- Use `set -euo pipefail` as first line after shebang

---

## Common Cross-Cutting Folders

These can be added to any archetype when relevant:

| Folder | When to include | Contents |
|--------|----------------|----------|
| `.github/workflows/` | GitHub-hosted CI | CI/CD pipeline YAML |
| `.woodpecker/` | Forgejo/Woodpecker CI | Pipeline configs |
| `.forgejo/` | Forgejo-specific | Issue templates, actions |
| `docs/adr/` | Architecture decisions exist | ADR-001.md, ADR-002.md... |
| `examples/` | Library with usage demos | Runnable example code |
| `benches/` | Performance-sensitive code | Benchmark suites |
| `fuzz/` | Fuzz testing (Rust/Go) | Fuzz targets and corpus |
| `assets/` | Static files needed at runtime | Images, templates, fonts |
| `configs/` | Multiple config variants | Per-env or per-target configs |
| `presets/` | Pre-built config bundles | Named config profiles |
| `.devcontainer/` | VS Code devcontainer | devcontainer.json + Dockerfile |
| `molecule/` | Ansible role testing | Molecule scenarios per role |
| `collections/` | Ansible Galaxy deps | requirements.yml |
| `flux-system/` | Flux GitOps bootstrap | Flux toolkit components |
| `components/` | Kustomize reusable pieces | Kustomize components |
| `completions/` | CLI with shell completions | bash, zsh, fish scripts |

---

## Scaffold Script

When generating a new project, create the structure with:

```bash
#!/bin/bash
set -euo pipefail

PROJECT="$1"
mkdir -p "$PROJECT"/{src,tests,docs,scripts}

# Meta files
touch "$PROJECT"/{README.md,LICENSE,.gitignore,CHANGELOG.md}

# Git init
cd "$PROJECT"
git init
echo "# $PROJECT" > README.md

echo "✅ Project scaffolded: $PROJECT/"
```

Adapt per archetype: add `Cargo.toml`, `go.mod`, `package.json`, etc. as needed.

---

## Output

- **Visualize**: always show the annotated tree in conversation
- **Scaffold**: create directories and placeholder files on the filesystem
- **Audit**: list issues + suggested fixes, then show the "ideal" tree side-by-side
- If the `readme` skill is also being used, feed the annotated tree into Tier 3 "Project Structure"
