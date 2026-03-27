# Project Type Adaptations

> **When to read:** During Step 3 after detecting the project type in Step 1, to apply type-specific README sections.

---

## Libraries (`rust-lib` / `go-lib` / `python-lib` / `js-lib`)
- **Tier 2 Quickstart**: package manager install + minimal code example with output
- **Tier 2 extra**: "API Overview" — top 3–5 public types/functions
- **Tier 3 extra**: Minimum Supported Version (MSRV / Go / Python)
- **Badge**: crates.io / pkg.go.dev / PyPI / npm

## CLI tools (`rust-cli` / `go-cli` / `python-cli`)
- **Tier 1**: terminal GIF or asciinema strongly recommended
- **Tier 2 Quickstart**: install method + one command with output
- **Tier 2 extra**: "Usage" with flags/subcommands table, "Configuration" if config file exists

## Infrastructure (`infra-terraform` / `infra-helm`)
- **Tier 1**: architecture diagram (mermaid or ASCII)
- **Tier 2 Quickstart**: apply/install + how to verify it works
- **Tier 2 extra**: Inputs/Outputs table (TF) or Values table (Helm), Requirements
- **Tier 3 extra**: State Management, Secrets Handling

## Backend apps (`app-backend`)
- **Tier 2 Quickstart**: `docker compose up` → curl that proves it works
- **Tier 2 extra**: API Endpoints summary table, Environment Variables table
- **Tier 3 extra**: Deployment, Monitoring, Database Migrations

## Containers (`container` / `podman-quadlet`)
- **Tier 2 Quickstart**: `podman run` / `docker run` with all required flags shown
- **Tier 2 extra**: Environment Variables, Volumes, Ports tables
- **Tier 2 extra for Quadlet**: show `systemctl --user start app.service` flow

## Kustomize / Flux (`kustomize` / `flux`)
- **Tier 1**: architecture diagram (what gets deployed where)
- **Tier 2 Quickstart**: `kubectl apply -k overlays/dev/` or `flux reconcile`
- **Tier 2 extra**: Overlays table (env → what changes), Prerequisites (cluster, Flux bootstrap)
- **Tier 3 extra**: "GitOps Workflow" (push → reconcile → verify), "Secrets Management"

## Ansible (`ansible`)
- **Tier 1**: what the playbooks configure + target infrastructure
- **Tier 2 Quickstart**: `ansible-playbook -i inventory/dev playbooks/site.yml`
- **Tier 2 extra**: "Roles" table (role → what it does), "Inventory Structure"
- **Tier 2 extra**: "Testing" if Molecule present (`molecule test`)
- **Tier 3 extra**: "Variables Precedence", "Vault Secrets", "CI Integration"

## Shell scripts (`shell-scripts`)
- **Tier 2 Quickstart**: `./bin/setup.sh` with expected output
- **Tier 2 extra**: "Scripts" table (script → purpose → args)
- **Tier 3 extra**: "Dependencies" (required tools), "Testing" (bats/shellspec)

## Monorepos
- **Tier 1**: overview of what's in the repo
- **Tier 2**: table of packages/services, one-liner each, links to sub-READMEs
- **Tier 3**: Workspace Setup, Inter-package Dependencies
