---
name: cli-git-conventional
description: >
  Enforce Conventional Commits v1.0.0, SemVer 2.0.0, branch naming, and human ghostwriter
  style on all git/jj operations. Zero AI markers. Use on 'commit', 'branch', 'tag',
  'release', 'changelog', 'semver', 'bump version', 'next version', 'CHANGELOG.md'.
  Also trigger proactively when the user shares code changes without mentioning commit.
context: fork
agent: general-purpose
allowed-tools:
  - Read
  - Bash
  - Grep
  - Glob
---

> **Optimization:** This skill uses on-demand loading. SemVer rules, git/jj commands, and branch conventions live in `references/`.

> **Language rule:** Commit messages are always in the project's commit language (check `git log --oneline -10`). If mixed, default to English. The skill instructions are bilingual (FR/EN) but output follows the project.

# Conventional Commits -- Ghostwriter

> "A commit message is a letter to your future self. Write it in your voice, not a machine's."

Reference: https://www.conventionalcommits.org/en/v1.0.0/

## Rule zero: Authorship

**NEVER add `Co-authored-by: Claude` or any AI trailer.** The commit belongs entirely to the person who asked. No `Co-authored-by`, `Generated-by`, `AI-assisted`, or equivalent trailers. Ever.

## Ghostwriter style

Claude is the **ghostwriter**. The message is signed by the user, written in their voice.

| Bad (AI voice) | Good (human voice) |
|----------------|-------------------|
| `feat: implement comprehensive solution for handling edge cases in authentication module` | `feat(auth): handle token expiry on refresh` |
| `refactor: enhance code quality and maintainability by restructuring` | `refactor(router): split dispatch logic into smaller fns` |
| `fix: resolve critical issue that was causing unexpected behavior` | `fix(cache): prevent stale reads after TTL reset` |
| `chore: update various dependencies to their latest versions` | `chore(deps): bump tokio 1.35 -> 1.37` |

**Banned words:** comprehensive, robust, leverage, utilize, enhance, streamline, facilitate, in order to, as part of, to ensure that.

**Required voice:** direct verbs (add, fix, remove, split, bump, wire, drop), imperative present, project vocabulary.

## Commit format

```
<type>[scope][!]: <description>

[body]

[footer(s)]
```

### Types

| Type | SemVer | Usage |
|------|--------|-------|
| `feat` | MINOR | New feature |
| `fix` | PATCH | Bug fix |
| `build` | -- | Build system, external deps |
| `chore` | -- | Maintenance, no functional impact |
| `ci` | -- | CI/CD configuration |
| `docs` | -- | Documentation only |
| `style` | -- | Formatting, whitespace (no logic change) |
| `refactor` | -- | Restructuring without feat or fix |
| `perf` | PATCH | Performance optimization |
| `test` | -- | Adding or modifying tests |
| `revert` | -- | Reverting a previous commit |

If a change spans multiple types, **split into multiple commits**.

### Description (subject line)

- Imperative present: `add`, `fix`, `remove` (never `added`, `fixes`, `removing`)
- No capital first letter: `feat: add retry logic` -- not `feat: Add retry logic`
- No trailing period
- Max **72 characters** (entire subject line including type)
- Precise and factual: **what** + optionally **why** in one phrase

### Body

- Separated from description by **one blank line** (mandatory)
- Explain the **why**, not the how (the code shows the how)
- Free paragraphs, wrap at 72 characters
- Can reference issues, tickets, PRs

### Footers

```
BREAKING CHANGE: <what breaks>
Refs: #123, #456
Closes: #789
```

No `Co-authored-by: Claude` or similar. Ever.

### Breaking changes

Signal with `!` before `:` and/or `BREAKING CHANGE:` footer:
```
feat(api)!: remove legacy v1 endpoints

BREAKING CHANGE: /api/v1/* routes no longer served. Migrate to /api/v2/*.
```

## Workflow

### Step 1 -- Analyze the diff

Before writing a commit message:
1. Read `git diff --staged` (or the changes the user shared)
2. Read `git log --oneline -5` for context and style
3. Identify: is this one logical change or multiple? If multiple, suggest splitting

### Step 2 -- Classify

Determine the type. When in doubt:
- Does it add user-visible functionality? -> `feat`
- Does it fix a bug? -> `fix`
- Does it change behavior without adding or fixing? -> `refactor`
- Does it only change tests? -> `test`
- Does it only change docs? -> `docs`
- Does it only change CI? -> `ci`
- Does it only bump deps? -> `chore(deps)` or `build(deps)`

### Step 3 -- Scope

Choose the most specific scope:
- Module name: `auth`, `router`, `cache`, `dlp`
- Layer: `api`, `db`, `ui`, `cli`
- Dependency: `deps`
- No scope if truly project-wide

### Step 4 -- Write

Apply ghostwriter rules:
1. Subject: imperative, lowercase, <=72 chars, no filler
2. Body (if needed): why this approach, not what the diff says
3. Footers: refs, breaking changes, closes
4. **No AI trailers**

### Step 5 -- Deliver

Use HEREDOC for multi-line messages:
```bash
git commit -m "$(cat <<'EOF'
fix(auth): handle expired tokens

Remove the silent swallow of TokenExpiredError. Callers now
receive a 401 with Retry-After header.

Refs: #88
EOF
)"
```

> Read `references/commands.md` for full git and jj command reference.

## SemVer and releases

> Read `references/semver.md` for SemVer 2.0.0 rules, tag commands, version bump resolution, and CHANGELOG format.

## Branch naming

> Read `references/branches.md` for branch naming conventions and examples.

## Anti-patterns

| Bad | Good |
|-----|------|
| `fix stuff` | `fix(cache): prevent stale entries after TTL expiry` |
| `WIP` | Atomic commits with proper messages |
| `feat: Added new feature` | `feat: add retry backoff on transient errors` |
| `Update` / `Changes` | Explicit message with type |
| Everything as `chore` | Use appropriate type |
| Scope too broad: `feat(app):` | Precise scope: `feat(router):` |
| 12 unrelated files in one commit | Atomic: one logical change per commit |
| `Co-authored-by: Claude` | No AI trailers |

## AI marker audit (history + files cleanup)

When invoked with `--audit-markers` or when the user asks to clean AI traces from a project:

### Step 1 -- Detect commit trailers

Scan for ALL known AI tool trailers and bot authors:

```bash
# Trailers (all AI tools)
git log --all --format="%H %s" --grep="Co-authored-by\|Co-Authored-By\|Generated by\|Generated with\|Generated-by\|AI-assisted" | \
  grep -iE "claude|copilot|opencode|codex|cursor|aider|gemini|devin|windsurf|anthropic|openai"

# Bot authors (Copilot, Devin, Jules)
git log --all --format="%H %an" | grep -iE "\[bot\]|copilot|devin-ai|jules"

# Aider author suffix
git log --all --format="%H %an" | grep "(aider)"

# PR footers in merge commits
git log --all --merges --format="%H %b" | grep -iE "claude.com/claude-code|cursor.com|Generated with|<!-- Cursor -->"
```

**Known trailers by tool:**

| Tool | Trailer / marker |
|------|-----------------|
| Claude Code | `Co-Authored-By: Claude <noreply@anthropic.com>` (with optional model name) |
| GitHub Copilot | Author: `github-copilot[bot]`, trailer: `Co-Authored-By: GitHub Copilot` |
| OpenAI Codex | `Co-Authored-By: Codex <noreply@openai.com>` |
| Aider | Author suffix `(aider)`, optional prefix `aider: ` in message |
| Devin | Author: `devin-ai[bot]`, footer: `Generated by Devin` |
| OpenCode | `Co-Authored-By: opencode <noreply@opencode.ai>` |
| Gemini/Jules | Author: Jules bot |
| Cursor | `<!-- Cursor -->` in PR descriptions, `cursor-` branch prefixes |

### Step 2 -- Detect AI config files

```bash
# Files that signal AI-assisted development
for f in CLAUDE.md GEMINI.md AGENTS.md llms.txt llms-full.txt \
         .cursorrules .cursorignore .cursorindexignore \
         .windsurfrules .codeiumignore \
         .aider.conf.yml .aider.chat.history.md .aider.input.history ".aider.tags.cache.v3" \
         .github/copilot-instructions.md .github/copilot-setup-steps.yml; do
  [ -f "$f" ] && echo "FOUND: $f"
done
[ -d .cursor ] && echo "FOUND: .cursor/"
[ -d .windsurf ] && echo "FOUND: .windsurf/"
[ -d .claude ] && echo "FOUND: .claude/"
[ -d .github/agents ] && echo "FOUND: .github/agents/"
[ -d .continue ] && echo "FOUND: .continue/"
[ -d .junie ] && echo "FOUND: .junie/"
```

### Step 3 -- Detect AI markers in .gitignore

Check if AI config files are already gitignored. If NOT, recommend adding them.

### Step 4 -- Report

```markdown
## AI Marker Audit -- {project}

### Commits: N trailers found
| Tool | Count | Branches |
| ... |

### Files: N AI config files found
| File | Tool | In .gitignore? |
| ... |

### Branch names: N AI-prefixed branches
| Branch | Tool |
| ... |
```

### Step 5 -- Propose cleanup options

| Option | Risk | How |
|--------|------|-----|
| **A. Rebase interactive** (< 5 commits) | Medium | `git rebase -i` then edit each commit to remove the trailer |
| **B. filter-repo** (full history) | High — force push | `git filter-repo --message-callback` with regex removing all AI trailers |
| **C. Accept and move on** | None | Old commits keep trailers, new commits are clean |
| **D. Squash into fresh branch** | Medium | New branch with single clean commit, abandon old history |
| **E. Absorb + delete files** | Low | Move AI file content to standard docs, delete AI files, add to .gitignore |

### Step 6 -- Recommend

| Situation | Options |
|-----------|---------|
| < 5 affected commits, local branch | **A** + **E** |
| Many commits, pre-audit | **B** + **E** (coordinate with team) |
| Shared branch, multiple contributors | **C** + **E** (don't rewrite shared history) |
| Fresh project | **D** + **E** |

### Step 7 -- Preventive .gitignore

Always recommend adding to `.gitignore`:

```gitignore
# AI tool config (not project documentation)
.cursor/
.cursorrules
.cursorignore
.cursorindexignore
.windsurf/
.windsurfrules
.codeiumignore
.aider*
.claude/
.continue/
.junie/
```

Note: `CLAUDE.md`, `GEMINI.md`, `AGENTS.md` are NOT gitignored — they are absorbed into standard docs by `cli-forge-doc` and then deleted.

**Never run filter-repo or force push without explicit user approval.**

## Dynamic Handoffs

| Condition detected | Recommend | Why |
|-------------------|-----------|-----|
| AI markers found in history | `/cli-forge-doc` | Absorb CLAUDE.md/AGENTS.md into standard docs |
| Commit messages reference issues | Project management (no skill) | Document as info |
| Breaking changes detected | `/cli-forge-readme` | Update README with migration notes |

**Rule:** Recommend, don't auto-execute.

## What this skill does NOT do

- **Does not run git commands autonomously** -- it writes messages, the user (or the calling skill) executes
- **Does not enforce commit hooks** -- it provides the message format, not pre-commit validation
- **Does not manage CI/CD** -- that's cli-forge-pipeline's job
- **Does not generate changelogs automatically** -- it provides the format, the user generates

## Integration with other cli-* skills

| Skill | Relationship |
|-------|-------------|
| `cli-cycle` | After fixes, cli-cycle should invoke cli-git-conventional for commit messages |
| `cli-forge-pipeline` | Pipeline may enforce commit format via CI hooks |
| `cli-audit-code` | Code audit may suggest refactors -- cli-git-conventional writes the commit |
| `cli-audit-code` | If C9 flags credential issues, cli-git-conventional writes `chore(auth):` for rotation commits |

## Checklist (internal, before delivering a message)

```
[ ] Type correct from allowed list
[ ] Scope relevant (or absent if not applicable)
[ ] ! and/or BREAKING CHANGE footer if breaking
[ ] Description: imperative present, lowercase, no period, <=72 chars
[ ] Body separated by blank line if present
[ ] Valid footers (token: value)
[ ] NO Claude / AI / co-author trailer
[ ] Atomic commit (one logical change)
[ ] Human voice (no banned words)
```
