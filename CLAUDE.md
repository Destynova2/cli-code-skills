# CLAUDE.md — cli-code-skills

## Skills installation

Skills are installed as **copies** in `~/.claude/skills/` for Claude Code,
`~/.codex/skills/` for Codex, and `~/.jcode/skills/` for Jcode (global,
available in all projects). Jcode is only touched when `~/.jcode` exists.
Source of truth is this repo (`~/workspace/cli-code-skills/`).

**After modifying a skill in this repo, copy it to every runtime:**
```bash
scripts/install_skills.sh
```

Or install only one runtime:
```bash
scripts/install_skills.sh --claude
scripts/install_skills.sh --codex
scripts/install_skills.sh --jcode
```

The installer removes each destination skill before copying it. That avoids stale
files when a reference is renamed or deleted.

The safe direction is repo → runtime. If a runtime copy was edited in place and
that edit is not yet in the repo, the installer **refuses** to overwrite it
(exit 3) rather than destroy it. It tracks this with a `.skills-manifest` written
into each runtime skills root at install time.

```bash
scripts/install_skills.sh --check          # report divergence, change nothing
scripts/install_skills.sh --pull-claude    # copy Claude in-place edits back into the repo
scripts/install_skills.sh --pull-codex     # copy Codex in-place edits back into the repo
scripts/install_skills.sh --pull-jcode     # copy Jcode in-place edits back into the repo
scripts/install_skills.sh --force          # overwrite even runtime-ahead edits
```

If you (or a session) edited a skill directly in `~/.claude/skills/`, run
`--pull-claude` to bring it back into the repo, review `git diff`, validate, and
commit — then a normal install re-syncs both runtimes.

**Shared files** (`gotchas.md` and `shared/`) are referenced by root `SKILL.md`
files as `../<name>` and by `references/` files as `../../<name>`.
The installer copies them to the skills root for every runtime.

Do NOT use symlinks — use real copies.

## Validation

Before committing or installing changes, run:

```bash
python3 scripts/validate_skills.py
```

The validator checks frontmatter, skill counts, README/CLAUDE inventories,
shared-file paths, and Claude/Codex-neutral `cli-cycle` orchestration.

It also enforces the Agent Skills spec limits: `name` <= 64 chars and
lowercase/digits/hyphens only, `description` <= 1024 chars (over the limit the
skill is rejected by the API). SKILL.md bodies over 500 lines are reported as
`WARN`, following Anthropic's progressive-disclosure guidance: move detail into
`references/` so the body stays cheap to load.

## Evaluations

`evals/` holds one JSON case per scenario plus the deliberately flawed fixtures
they point at. There is no scored runner: grading a skill run needs a model in
the loop. What is automated keeps the cases honest.

```bash
python3 scripts/check_evals.py
```

It rejects a case whose skill or fixture does not exist, or that cites a
dimension ID the skill never defines. See `evals/README.md`. Fixtures are
broken on purpose and must not be repaired.

## Current skills (38)

**Audit (12):** cli-audit-code, cli-audit-data, cli-audit-doc, cli-audit-drift, cli-audit-hanoi, cli-audit-review, cli-audit-shell, cli-audit-sync, cli-audit-tangle, cli-audit-test, cli-audit-wizard, cli-audit-xray

**Cycle (1):** cli-cycle

**Forge (23):** cli-forge-arch, cli-forge-chef, cli-forge-choice-ux, cli-forge-data, cli-forge-demo, cli-forge-doc, cli-forge-github, cli-forge-hld, cli-forge-influence, cli-forge-infra, cli-forge-lld, cli-forge-nego, cli-forge-oci-rootless, cli-forge-perf, cli-forge-pipeline, cli-forge-plume, cli-forge-prez, cli-forge-profile, cli-forge-quorum, cli-forge-readme, cli-forge-resilience, cli-forge-schema, cli-forge-tree

**Git (1):** cli-git-conventional

**Watermark (1):** cli-watermark

## Git identity

Use `clement <cliard@a00.fr>` for this repo. Never use naval-group email.

## Commit style

This repo uses the `cli-git-conventional` skill. Never add `Co-Authored-By` trailers.
Follow Conventional Commits v1.0.0 spec. Ghostwriter style — human voice, no AI markers.
