# Tooling Ladder -- Complexity-Adapted Tool Selection

> **When to read:** Step 6 (Assess tooling fit). Determines whether a script uses the right tool for its complexity level.

---

## Core Principle

> **L'outil doit correspondre a la complexite du probleme, pas l'inverse.**
> Un sed suffit pour remplacer un port. Un kustomize est necessaire pour gerer 15 overlays.
> Utiliser kustomize pour un port, c'est de l'over-engineering. Utiliser sed pour 15 overlays, c'est du sabotage.

## The Tooling Ladder

Each rung handles a specific complexity band. Going UP adds capability but also dependencies, learning curve, and operational overhead. Going DOWN is simpler but loses structure.

```
Rung 5: Full templating engine (Helm, Jsonnet, CUE, Dhall)
  |  When: 10+ environments, conditionals, loops, type checking
  |  Deps: language runtime, chart repos, schema validation
  |
Rung 4: Kustomize / overlay system
  |  When: 3+ environments, same base with variations, strategic merge patches
  |  Deps: kustomize binary (built into kubectl)
  |
Rung 3: Structured parser (yq, jq, xmlstarlet, tomlq)
  |  When: YAML/JSON/XML manipulation, key path access, array operations
  |  Deps: single binary (yq/jq), understands the format grammar
  |
Rung 2: sed/awk with anchored patterns
  |  When: simple text replacement, key=value files, 1-3 substitutions
  |  Deps: none (POSIX builtins)
  |
Rung 1: Shell builtins (parameter expansion, printf)
  |  When: variable defaults, string manipulation, simple conditionals
  |  Deps: none
  |
Rung 0: Hardcode it
   When: truly static values that never change across environments
```

## Decision Tree

```
Q1: Is the data structured (YAML, JSON, XML, TOML)?
  NO  -> Q2
  YES -> Q3

Q2: How many substitutions?
  1-3 simple key=value -> Rung 1-2 (shell builtins or sed)
  4+ or nested         -> Rung 3 (yq/jq for the format)

Q3: How many environments/variations?
  1 (single target)     -> Rung 3 (yq/jq: parse, modify, emit)
  2-3 (dev/staging/prod) -> Rung 4 (kustomize: base + overlays)
  4+ with conditionals   -> Rung 5 (Helm/Jsonnet: full templating)

Q4: Does the script GENERATE structured data?
  YES, from variables -> Rung 3+ (NEVER build YAML with echo/cat/heredoc)
  YES, static template -> Rung 2 (sed on a .tmpl file is OK for 1-3 vars)
  NO                  -> N/A
```

## Common Mismatches (Anti-Patterns)

### Over-Engineering (tool too complex for the problem)

| Pattern | Problem | Better approach |
|---------|---------|----------------|
| Kustomize for a single YAML with 2 variables | Adds kustomization.yaml, base/, overlays/ for what sed does in one line | `sed "s/__PORT__/${PORT}/" template.yaml` |
| Helm chart for a single-pod deployment | Chart.yaml, values.yaml, templates/ for a 30-line manifest | Plain YAML + kustomize if 2+ envs |
| Jsonnet/CUE for a docker-compose.yml | Learning curve exceeds the problem complexity | yq for dynamic values, plain YAML for static |
| jq to extract a single field from /etc/os-release | os-release is key=value, not JSON | `source /etc/os-release; echo "${VERSION_ID}"` |

### Under-Engineering (tool too simple for the problem)

| Pattern | Problem | Better approach |
|---------|---------|----------------|
| `sed 's/old/new/'` on YAML with nested keys | Breaks on indentation, comments, multi-line values | `yq '.spec.containers[0].image = "new"' manifest.yaml` |
| `awk '{print $2}'` on JSON | Positional parsing breaks on format changes | `jq '.key'` |
| Heredoc with `__PLACEHOLDER__` for 10+ variables | Unmaintainable, no validation, injection risk | Kustomize overlays or Helm values |
| `grep + cut` to parse YAML secrets | Breaks on quoted values, multi-line, special chars | `yq '.data.password' secret.yaml` |
| `echo "key: ${value}"` to build YAML | No escaping for special chars (:, #, -, [) | `yq -n ".key = \"${value}\""` |

## Scoring Impact

Tooling mismatches don't directly impact the SQI score (they're not a shell quality issue per se), but they appear in the **Tooling Recommendations** section of the report.

| Mismatch type | Report section | Severity |
|--------------|---------------|----------|
| Under-engineering on structured data | Tooling Recommendations | Major |
| Over-engineering on simple substitution | Tooling Recommendations | Minor |
| Correct tool for complexity | (not flagged) | -- |

## Real-World Example: MariaDB Deployment

```
File: mariadb-play-kube.yaml.tmpl
  - 15 __PLACEHOLDER__ variables
  - Full Kubernetes Pod manifest
  - Rendered by sed in a shell script

Assessment:
  Q1: Structured? YES (YAML Pod manifest)
  Q3: How many environments? 2 (VM-specific + generic)
  Decision: Rung 4 (kustomize)
    - base/mariadb-play-kube.yaml (no placeholders, real defaults)
    - overlays/vm/kustomization.yaml (patches for VM-specific values)

  If kustomize is not available:
    Rung 3 (yq)
    - yq '.spec.containers[0].resources.limits.cpu = env(CPU_LIMIT)' base.yaml

  Rung 2 (sed) is WRONG here because:
    - 15 variables = too many sed expressions
    - YAML structure means sed can break indentation
    - No validation that all placeholders were replaced
```

## Quick Reference Card

| Complexity signal | Rung | Tool |
|------------------|------|------|
| Static value, never changes | 0 | Hardcode |
| `${VAR:-default}`, simple string ops | 1 | Shell builtins |
| 1-3 `__PLACEHOLDER__` in plain text or `.conf` | 2 | `sed` |
| Modify YAML/JSON keys, extract values, filter arrays | 3 | `yq` / `jq` |
| Same manifest, 2-4 environment variations | 4 | Kustomize |
| 5+ envs, conditional logic, loops, type safety | 5 | Helm / Jsonnet / CUE |

## Dependencies & Availability

| Tool | Install | Size | Availability in containers |
|------|---------|------|---------------------------|
| sed/awk | POSIX built-in | 0 | Always available |
| yq | Single binary (~10 MB) | Small | Easy to add |
| jq | Single binary (~1 MB) | Small | Easy to add |
| kustomize | Single binary (~30 MB) or `kubectl kustomize` | Medium | Built into kubectl |
| Helm | Single binary (~50 MB) | Medium | Separate install |
| Jsonnet/CUE | Language runtime | Large | Separate install |
