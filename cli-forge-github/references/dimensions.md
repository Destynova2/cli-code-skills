# Dimensions — Detailed Scoring Criteria

> **When to read:** Phase 1, auditing each dimension.

---

## D1 — Ruleset ↔ CI Alignment (max 15)

**Check:** For each ruleset, extract required status checks. For each required check, verify it ALWAYS emits a status (even when path pruning skips the job).

```bash
# Extract required checks from all rulesets
gh api repos/${REPO}/rulesets | jq -r '.[].id' | while read id; do
  gh api repos/${REPO}/rulesets/${id} | jq -r '.rules[] |
    select(.type == "required_status_checks") |
    .parameters.required_status_checks[].context'
done
```

Then for each required check name, grep the CI YAML:
- Is it a real job name? Or a summary job?
- Does the job have `rules:` or `paths:` filters that could skip it?
- If skippable, does a summary job emit the status anyway?

| Score | Criteria |
|-------|----------|
| 0 | Rulesets require checks that can be skipped by path pruning (eternal pending) |
| 5 | Rulesets require checks, but all are always-run jobs |
| 10 | Single summary job as required check, handles skipped jobs |
| 15 | Summary job + documentation comment in CI YAML explaining the design |

---

## D2 — Branch Hygiene (max 10)

**Check:**

```bash
# Remote branches not in open PRs
OPEN_BRANCHES=$(gh pr list --repo ${REPO} --state open --json headRefName -q '.[].headRefName')
gh api repos/${REPO}/branches --paginate | jq -r '.[].name' | while read branch; do
  echo "$OPEN_BRANCHES" | grep -q "^${branch}$" || echo "ORPHAN: ${branch}"
done
```

Also check:
- Local branches with no remote tracking
- Dangling worktrees (`git worktree list`)
- `delete_branch_on_merge` setting

| Score | Criteria |
|-------|----------|
| 0 | 5+ orphan branches, no auto-delete |
| 4 | 1-4 orphan branches |
| 7 | 0 orphans, but auto-delete not enabled |
| 10 | 0 orphans, auto-delete enabled, 0 dangling worktrees |

---

## D3 — PR Lifecycle (max 10)

**Check:**

```bash
gh pr list --repo ${REPO} --state open --json number,title,createdAt,isDraft,autoMergeRequest
```

| Score | Criteria |
|-------|----------|
| 0 | 3+ PRs > 14 days old, or auto-merge enabled but stuck |
| 4 | 1-2 PRs > 7 days old |
| 7 | No stale PRs, but auto-merge not configured |
| 10 | No stale PRs, auto-merge working, drafts are recent |

---

## D4 — Release Flow (max 15)

**Check:**
- Is release-plz configured? (`release-plz.toml` or `Cargo.toml [workspace.metadata.release-plz]`)
- Any release PR in conflict?
- sync-main PR in conflict?
- Latest tag matches latest release?
- develop ahead of main by more than 1 release?

| Score | Criteria |
|-------|----------|
| 0 | Release PR in conflict, or sync-main diverged > 2 releases, or multi-agent push rejected by branch protection |
| 5 | Release PR clean, sync-main has minor conflicts |
| 10 | Release flow working, sync-main automated |
| 13 | Release-plz + sync-main + tag + GitHub Release all in sync |
| 15 | All in sync + multi-agent workflow uses PR mode on protected branches (G26) |

**Also check:** Do any rulesets on develop/main require PRs? If yes, verify that cli-forge-chef's merge protocol is in PR mode. If a recent sprint had `GH006: Protected branch update failed` errors → flag as critical and recommend G26 fix.

---

## D5 — CI Health (max 15)

**Check:**

```bash
gh run list --repo ${REPO} --limit 10 --json status,conclusion,name
```

Classify failures:
- **Real:** code/config bug → counts against score
- **Transient:** rate limit, OOM, network → rerun, doesn't count

| Score | Criteria |
|-------|----------|
| 0 | 3+ real failures in last 10 runs |
| 5 | 1-2 real failures |
| 10 | 0 real failures, some transient (rerun resolved) |
| 15 | All green, no transient issues |

---

## D6 — Permissions & Scopes (max 10)

**Check:**
- CODEOWNERS file exists and is valid?
- CLA configured? (`cla-check` in PR checks)
- Token scopes appropriate? (no `workflow` scope lingering)

| Score | Criteria |
|-------|----------|
| 0 | No CODEOWNERS, no CLA, token has excessive scopes |
| 5 | CODEOWNERS exists, CLA configured |
| 10 | CODEOWNERS valid, CLA passing, token scopes minimal |

---

## D7 — Default Branch Config (max 10)

**Check:**

```bash
gh api repos/${REPO} | jq '{
  delete_branch_on_merge,
  allow_squash_merge,
  allow_merge_commit,
  allow_rebase_merge,
  squash_merge_commit_title,
  squash_merge_commit_message
}'
```

| Score | Criteria |
|-------|----------|
| 0 | No merge strategy enforced, branches not auto-deleted |
| 5 | Squash merge enabled |
| 8 | Squash + auto-delete |
| 10 | Squash + auto-delete + meaningful squash commit format |

---

## D8 — Workflow Hygiene (max 15)

**Check:**
- Any `@v3` actions that have `@v4`? (deprecated)
- Any `@master` or `@main` action references? (unpinned)
- Any unused workflow files? (never triggered)
- Action versions pinned to SHA?

```bash
# Find unpinned actions
grep -rn 'uses:' .github/workflows/ | grep -vE '@[a-f0-9]{40}' | grep -vE '@v[0-9]'
```

| Score | Criteria |
|-------|----------|
| 0 | Actions on `@master`, deprecated versions, unused workflows |
| 5 | All actions on version tags (@v4) but not SHA-pinned |
| 10 | All actions SHA-pinned, no deprecated |
| 15 | SHA-pinned + Dependabot/Renovate configured for action updates |
