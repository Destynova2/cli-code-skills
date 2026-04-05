# Branch Naming Conventions

> **When to read:** When creating branches, reviewing branch names, or setting up workflows.

## Pattern

```
<type>/<description-kebab-case>
```

## Rules

- Lowercase only
- Kebab-case (`-`), no `_` or spaces
- `/` as type/description separator
- Short explicit description (3-5 words max)
- No ticket number alone -- prefer a readable description
- Optional: prefix with ticket ID: `feat/GH-123-add-dlp-middleware`

## Examples

```
feat/add-dlp-middleware
fix/auth-token-expiry
chore/update-rust-deps
ci/add-arm64-runner
docs/update-api-reference
refactor/split-router-module
feat/GH-42-fan-out-racing
```

## Special branches

| Branch | Usage |
|--------|-------|
| `main` | Primary branch, production-ready |
| `develop` | Integration (if gitflow workflow) |
| `release/x.y.z` | Release preparation |
| `hotfix/<description>` | Critical fix on main |
