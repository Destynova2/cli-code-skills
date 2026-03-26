# Naming Conventions & Structural Rules

> **When to read:** During audit mode (Step 2) when checking naming consistency, depth violations, or cross-cutting folder usage.

---

## Case & Separators

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

---

## Naming Principles

| Principle | Good | Bad | Why |
|-----------|------|-----|-----|
| **Descriptive but short** | `docs/` | `documentation-files/` | 3-8 chars ideal for top-level |
| **Role-obvious** | `scripts/` | `scr/` | No abbreviation puzzles |
| **No redundancy** | `src/auth/` | `src/auth-module/` | Parent gives context |
| **No version in names** | `api/` | `api-v2/` | Use git branches/tags |
| **Plural for collections** | `tests/`, `scripts/` | `test/`, `script/` | Unless ecosystem convention says otherwise |

---

## Depth Rule

**Max 3 levels before you need a good reason.**

```
src/api/handlers/          ← 3 levels, fine
src/api/handlers/v2/users/ ← 5 levels, rethink your structure
```

If you need 4+ levels, consider: is this a separate package/crate/module?

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
