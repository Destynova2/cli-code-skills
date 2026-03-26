# Archetype Templates — Full Project Structures

> **When to read:** During Step 2 (Generate mode) when scaffolding a new project, or when auditing an existing project against its archetype.

---

## `rust` (lib or CLI)

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

---

## `go` (lib or CLI)

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

---

## `python` (lib or CLI)

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

---

## `js-ts` (lib, CLI, or app)

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

---

## `terraform`

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

---

## `helm`

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

---

## `app-backend` (Docker Compose based)

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

---

## `monorepo`

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

---

## `oci-container` (Podman / Docker / OCI)

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

---

## `kustomize` / `flux` (GitOps Kubernetes)

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

---

## `ansible` (with optional Molecule)

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

---

## `podman-quadlet` (systemd integration)

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

---

## `shell-scripts` (Bash / POSIX sh)

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

echo "Project scaffolded: $PROJECT/"
```

Adapt per archetype: add `Cargo.toml`, `go.mod`, `package.json`, etc. as needed.
