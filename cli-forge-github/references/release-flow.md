# Release Flow — Dev to Prod Pipeline Health

> **When to read:** D4 audit (release flow dimension) and when diagnosing release pipeline issues.

---

## The ideal flow (github-flow-develop model, e.g., grob)

```
Commis pushes → PR → CI → auto-merge to develop
                                ↓
                    release-plz detects fix/feat commits
                                ↓
                    release-plz creates Release PR (version bump + changelog)
                                ↓
                    Release PR auto-merges → release-plz creates tag vX.Y.Z
                                ↓
                    Tag push triggers sync-main.yml
                                ↓
                    sync-main creates PR develop → main + enables auto-merge
                                ↓
                    CI passes → auto-merge → main catches up
                                ↓
                    absorb job: main → develop (prevents drift)
```

**Total expected time:** 15-30 min from commis PR merge to main updated.

## Where it breaks (known failure modes)

### FM1 — Release-plz PR conflicts on CHANGELOG.md

**Cause:** Multiple PRs merge to develop between release-plz runs. Each merge adds to CHANGELOG, but release-plz's PR is based on an older develop → conflict.

**Symptom:** Release PR shows "This branch has conflicts" forever.

**Fix (automated):** In `auto-merge-release.yml`:
```yaml
# If release PR has conflicts, close it. release-plz will recreate.
- name: Close conflicting release PRs
  run: |
    gh pr list --label release --state open --json number,mergeable \
      --jq '.[] | select(.mergeable == "CONFLICTING") | .number' \
    | while read pr; do
        gh pr close "$pr" --comment "Conflicts detected. release-plz will recreate."
      done
```

**Prevention in cli-forge-chef:** The Sous-Chef should merge one PR at a time and wait for release-plz to process before merging the next. This reduces the window for conflicts.

### FM2 — sync-main PR has merge conflicts

**Cause:** develop and main diverged (usually because CHANGELOG.md or Cargo.lock differ).

**Symptom:** sync-main PR shows conflicts on CHANGELOG.md, Cargo.toml, Cargo.lock.

**Fix:** sync-main.yml already handles this with `git merge origin/main --no-edit -X ours` (develop wins). If it still conflicts, the absorb job may be out of order.

**Prevention:** The absorb job (main → develop) after each sync prevents drift. Verify it runs successfully.

### FM3 — release-plz doesn't detect commits

**Cause:** Commits don't match the path filters in release-plz.yml (`src/**`, `Cargo.toml`, `Cargo.lock`). Example: a fix in `.github/workflows/` or `docs/` won't trigger a release.

**Symptom:** Commits on develop, no Release PR appears.

**Fix:** This is by design — only code changes trigger releases. Doc/CI changes ride the next code release.

### FM4 — auto-merge not enabled on release or sync PRs

**Cause:** `gh pr merge --auto` failed silently, or the repo doesn't have auto-merge enabled.

**Symptom:** PRs are mergeable (CI green, no conflicts) but nobody merges them.

**Fix:** Verify `allow_auto_merge` is enabled on the repo:
```bash
gh api repos/${REPO} --jq '.allow_auto_merge'
# If false:
gh api repos/${REPO} --method PATCH -f allow_auto_merge=true
```

### FM5 — CI fails on the release/sync PR

**Cause:** The release PR bumps `Cargo.toml` version, which changes the binary version string, which may fail a test that hardcodes the version.

**Symptom:** CI fails on the release PR with a test assertion error.

**Fix:** Tests should never hardcode the version. Use `env!("CARGO_PKG_VERSION")` or read from Cargo.toml at runtime.

### FM6 — Tag push doesn't trigger sync-main

**Cause:** The tag was pushed with a token that doesn't trigger workflow_dispatch (e.g., `GITHUB_TOKEN` instead of a PAT).

**Symptom:** Tag exists, but no sync-main PR appears.

**Fix:** release-plz must use a PAT (`RELEASE_PLZ_TOKEN`) not `GITHUB_TOKEN` for the tag push. Check:
```bash
gh api repos/${REPO}/actions/runs --jq '.workflow_runs[] | select(.event == "push" and .head_branch | startswith("v")) | .id' | head -3
```

## Pre-commit ↔ CI alignment audit

### What local prek should mirror from CI

| CI job | Local prek equivalent | Gap? |
|---|---|---|
| Rustfmt | `cargo fmt` (pre-commit) | ✅ No gap |
| Clippy (ubuntu) | `cargo clippy` (pre-commit) | ✅ No gap |
| Clippy (macos/windows) | — | ⚠️ Can't run locally on Linux. Acceptable gap. |
| Security Audit | `cargo audit` (pre-push) | ⚠️ Advisories change between push and CI. Acceptable. |
| Gitleaks | `gitleaks protect` (pre-commit) | ✅ No gap |
| Cargo Deny | `cargo deny` (pre-push) | ✅ No gap |
| Doc Coverage | doc coverage check (pre-push) | ✅ No gap |
| Unused Deps | `cargo machete` (pre-push) | ✅ No gap |
| Test (ubuntu) | `cargo nextest run` (pre-push) | ✅ No gap |
| Test (macos/windows) | — | ⚠️ Can't run locally. Acceptable. |
| Feature Powerset | — | ⚠️ Too slow for local. CI-only is correct. |
| Mutation Testing | — | ⚠️ Way too slow for local. CI-only is correct. |
| Coverage | — | ⚠️ Requires llvm-tools, slow. CI-only acceptable. |
| Container image | — | ⚠️ Requires podman/docker. Optional local check. |
| E2E tests | — | ⚠️ May require running services. CI-only acceptable. |
| Validate CI YAML | — | 🔴 **GAP**: CI YAML syntax not checked locally |

### Recommended additions to prek

```toml
# Validate CI YAML syntax locally (catches yaml errors before push)
[[repos.hooks]]
id = "validate-ci-yaml"
name = "validate CI YAML"
language = "system"
entry = "sh"
args = ["-c", "for f in .github/workflows/*.yml; do python3 -c \"import yaml; yaml.safe_load(open('$f'))\" || exit 1; done"]
files = '\\.github/workflows/.*\\.yml$'
pass_filenames = false
stages = ["pre-push"]
```

### What should NOT be in prek

- **Feature powerset** — 4+ minutes, frustrates devs
- **Mutation testing** — 20+ minutes, CI-only
- **Multi-OS tests** — impossible on Linux
- **Coverage** — slow, CI-only
- **Container build** — needs docker daemon, optional

## Release flow health score

| Check | Points | How |
|---|---|---|
| release-plz configured + working | 3 | Check `release-plz.toml` exists + last Release PR < 7 days old |
| sync-main workflow exists + runs | 3 | Check `sync-main.yml` + last run succeeded |
| absorb job keeps develop 0-behind-main | 3 | `git rev-list --count origin/develop..origin/main` == 0 |
| auto-merge enabled on repo | 2 | `gh api repos/${REPO} --jq '.allow_auto_merge'` == true |
| PAT used (not GITHUB_TOKEN) for tags | 2 | release-plz uses `RELEASE_PLZ_TOKEN` |
| No stale release PRs | 2 | 0 open PRs with label `release` older than 24h |
| prek covers all fast CI checks | 3 | fmt + clippy + test + deny + audit + gitleaks + machete + doc-coverage |
| Total | **18** | |
