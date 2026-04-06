# CLAUDE.md — cli-code-skills

## Skills installation

Skills are installed as **copies** in `~/.claude/skills/` (global, available in all projects).
Source of truth is this repo (`~/workspace/cli-code-skills/`).

**After modifying a skill in this repo, copy it to `~/.claude/skills/` :**
```bash
cp -r cli-audit-wizard ~/.claude/skills/cli-audit-wizard
```

Or copy all skills at once:
```bash
for d in cli-*/; do cp -r "$d" ~/.claude/skills/"${d%/}"; done
```

Do NOT use symlinks — use real copies.

## Current skills (20)

**Audit (10):** cli-audit-code, cli-audit-credential, cli-audit-doc, cli-audit-drift, cli-audit-shell, cli-audit-sync, cli-audit-tangle, cli-audit-test, cli-audit-wizard, cli-cycle

**Forge (10):** cli-forge-arch, cli-forge-boss, cli-forge-doc, cli-forge-hld, cli-forge-infra, cli-forge-lld, cli-forge-pipeline, cli-forge-readme, cli-forge-schema, cli-forge-tree

**Git (1):** conventional-commits

## Git identity

Use `clement <cliard@a00.fr>` for this repo. Never use naval-group email.

## Commit style

This repo uses the `conventional-commits` skill. Never add `Co-Authored-By` trailers.
Follow Conventional Commits v1.0.0 spec. Ghostwriter style — human voice, no AI markers.
