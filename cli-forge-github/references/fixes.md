# Fix Catalog — GitHub Repository Issues

> **When to read:** Phase 2, fixing detected issues.

---

## F1 — Ruleset requires skippable checks

**Detect:**
```bash
# Get required check names from rulesets
REQUIRED=$(gh api repos/${REPO}/rulesets | jq -r '.[].id' | while read id; do
  gh api repos/${REPO}/rulesets/${id} 2>/dev/null | jq -r '
    .rules[]? | select(.type == "required_status_checks") |
    .parameters.required_status_checks[]?.context'
done | sort -u)

# For each required check, see if it can be skipped
for check in $REQUIRED; do
  # Check if this job name has path filters in the CI YAML
  if grep -A20 "name:.*${check}\|${check}:" .github/workflows/*.yml | grep -q "paths:\|changes:"; then
    echo "VULNERABLE: ${check} can be skipped by path pruning"
  fi
done
```

**Fix:**
```bash
RULESET_ID=$(gh api repos/${REPO}/rulesets | jq '.[] | select(.name == "Protect develop") | .id')

# Read current ruleset
gh api repos/${REPO}/rulesets/${RULESET_ID} > /tmp/ruleset.json

# Replace individual checks with summary job
jq '.rules |= map(
  if .type == "required_status_checks" then
    .parameters.required_status_checks = [{"context": "Required checks"}]
  else . end
)' /tmp/ruleset.json | gh api repos/${REPO}/rulesets/${RULESET_ID} --method PUT --input -
```

**Post-fix:** Add comment in ci.yml explaining the design.

---

## F2 — Release PR in conflict (release-plz)

**Detect:**
```bash
gh pr list --repo ${REPO} --label release --json number,mergeable -q '.[] | select(.mergeable == "CONFLICTING")'
```

**Fix:** Close the conflicting PR. release-plz recreates on next develop push.
```bash
gh pr close ${PR} --repo ${REPO} --comment "Conflicts with develop. release-plz will recreate on next push."
```

**Alternative (if you want to keep the PR):** Rebase the release branch on develop:
```bash
git fetch origin ${RELEASE_BRANCH} develop
git checkout ${RELEASE_BRANCH}
git rebase origin/develop
git push origin ${RELEASE_BRANCH} --force-with-lease
```

---

## F3 — Orphan branches

**Detect:**
```bash
# Get all remote branches
ALL_BRANCHES=$(gh api repos/${REPO}/branches --paginate -q '.[].name')

# Get branches with open PRs
PR_BRANCHES=$(gh pr list --repo ${REPO} --state open -q '.[].headRefName')

# Protected branches
PROTECTED="main develop"

# Orphans = all - PRs - protected
for branch in $ALL_BRANCHES; do
  echo "$PR_BRANCHES $PROTECTED" | grep -qw "$branch" || echo "ORPHAN: $branch"
done
```

**Fix:**
```bash
for branch in ${ORPHAN_BRANCHES}; do
  gh api -X DELETE repos/${REPO}/git/refs/heads/${branch} 2>/dev/null && echo "Deleted: ${branch}"
done
```

**Prevention:**
```bash
gh api repos/${REPO} --method PATCH -f delete_branch_on_merge=true
```

---

## F4 — Transient CI failure

**Detect:** Check the error message of failed jobs:
```bash
RUN_ID=$(gh run list --repo ${REPO} --status failure --limit 1 -q '.[0].databaseId')
gh api repos/${REPO}/actions/runs/${RUN_ID}/jobs | jq -r '
  .jobs[] | select(.conclusion == "failure") | .id' | while read job_id; do
    gh api repos/${REPO}/actions/jobs/${job_id}/logs 2>/dev/null | grep -i "rate limit\|timeout\|OOM\|connection refused\|stepsecurity" && echo "TRANSIENT: job ${job_id}"
done
```

**Fix:**
```bash
gh run rerun ${RUN_ID} --repo ${REPO} --failed
```

If the run is still in progress, wait for it to finish first:
```bash
gh run watch ${RUN_ID} --repo ${REPO} --exit-status || gh run rerun ${RUN_ID} --repo ${REPO} --failed
```

---

## F5 — Token lacks workflow scope

**Detect:**
```bash
gh auth status 2>&1 | grep -q "workflow" && echo "SCOPE: workflow is active" || echo "SCOPE: workflow NOT active"
```

**Fix (temporary elevation):**
```bash
# Add scope
gh auth refresh -s workflow

# Do the push
git push origin ${BRANCH}

# Remove scope immediately
gh auth refresh
```

**Verify removal:**
```bash
gh auth status 2>&1 | grep "scope" | grep -v "workflow" && echo "Clean"
```

---

## F6 — sync-main conflicts

**Detect:**
```bash
gh pr list --repo ${REPO} --label sync-main --json number,mergeable -q '.[] | select(.mergeable == "CONFLICTING")'
```

**Fix:** Recreate from develop, resolve with develop winning:
```bash
git fetch origin develop main
git checkout -B sync-main-v${VERSION} origin/develop
git merge origin/main --no-edit -X theirs  # develop wins on conflicts
git push origin sync-main-v${VERSION} --force-with-lease
```

---

## F7 — Stale PRs

**Detect:**
```bash
# PRs open > 14 days, not draft, not labeled "keep-open"
gh pr list --repo ${REPO} --state open --json number,title,createdAt,isDraft,labels -q '
  .[] | select(
    .isDraft == false and
    (.labels | map(.name) | index("keep-open") == null) and
    (now - (.createdAt | fromdateiso8601) > 14*86400)
  ) | "\(.number) \(.title) (age: \((now - (.createdAt | fromdateiso8601)) / 86400 | floor)d)"'
```

**Fix:** Close with a comment:
```bash
gh pr close ${PR} --repo ${REPO} --comment "Closing stale PR (> 14 days). Reopen if still needed."
```

---

## F8 — Unpinned or deprecated actions

**Detect:**
```bash
# Find actions not pinned to SHA
grep -rn 'uses:' .github/workflows/*.yml | grep -vE '@[a-f0-9]{40}' | while read line; do
  echo "UNPINNED: $line"
done

# Find deprecated action versions (v3 when v4 exists)
grep -rn 'uses:.*@v3' .github/workflows/*.yml | while read line; do
  echo "POSSIBLY DEPRECATED: $line"
done
```

**Fix:** Pin to SHA (get the SHA for the tag):
```bash
# Example: actions/checkout@v4 → SHA
SHA=$(gh api repos/actions/checkout/git/refs/tags/v4 -q '.object.sha' 2>/dev/null)
# Replace in CI YAML: actions/checkout@v4 → actions/checkout@${SHA}
```

**Note:** Before bumping `@v3` → `@v4`, ALWAYS verify the v4 tag exists (see gotchas.md in cli-forge-pipeline).
