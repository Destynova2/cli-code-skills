# Git & jj Commands -- Cheat Sheet

> **When to read:** When executing git or jj operations. Use project-appropriate commands.

---

## Git

### Commit
```bash
git add -p                          # interactive staging, atomic commit
git commit -m "feat(scope): description"

# With body + footer (use HEREDOC for multi-line)
git commit -m "$(cat <<'EOF'
fix(auth): handle expired tokens

Remove the silent swallow of TokenExpiredError. Callers now
receive a 401 with Retry-After header.

Refs: #88
EOF
)"
```

### Amend (before push only)
```bash
git commit --amend --no-edit        # add forgotten files
git commit --amend -m "new message" # rewrite message
```

### Rebase
```bash
git rebase -i HEAD~3                # rewrite last 3 commits
# editor commands: reword, squash, fixup, drop
```

### Branch
```bash
git switch -c feat/add-dlp-middleware
git push -u origin feat/add-dlp-middleware
```

### Pre-commit checks
```bash
git diff --staged                   # inspect what's going
git log --oneline -10               # history context
```

---

## jj (Jujutsu)

### Describe
```bash
jj describe -m "feat(scope): description"

# Multi-line
jj describe -m "fix(auth): handle expired tokens

Remove the silent swallow of TokenExpiredError. Callers now
receive a 401 with Retry-After header.

Refs: #88"
```

### New commit
```bash
jj new -m "feat(scope): description"
```

### Branches (bookmarks)
```bash
jj bookmark create feat/add-dlp-middleware
jj bookmark set feat/add-dlp-middleware
jj git push --bookmark feat/add-dlp-middleware
```

### Edit past commit
```bash
jj edit <rev>                       # move to target commit
jj describe -m "new message"       # rewrite
jj new @+                           # return to descendant
```

### Squash
```bash
jj squash                           # merge into parent
jj squash --into <rev>              # merge into specific commit
```
