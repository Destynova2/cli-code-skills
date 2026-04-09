# Template — Shared State

Generate this file at `{project}/.claude/shared-state.md`.
Replace `{variables}` with the project values.

---

```markdown
# {Project Name} Shared State — Session {session_name}

> Shared memory between all parallel Claude Code sessions.
> **BOSS**: only agent allowed to write in "Valid merges" and "Green light".
> **WORKERS**: read this file BEFORE coding. Write in "In progress" and "Done".
> **EVERYONE**: re-read this file after every merge to see what changed.

## Green light — Resolved dependencies

> The boss writes here when a merge is validated and dependents can continue.
> Workers: if your task depends on another, wait until it appears here.

| Merged branch | Tests | CI | Date | Unblocks |
|---------------|-------|----|------|----------|

## In progress

| Worktree | Branch | Task | Files touched | Started |
|----------|--------|------|----------------|---------|

## Done (awaiting merge)

| Worktree | Branch | Task | Result | Tests | Files modified | Date |
|----------|--------|------|--------|-------|-----------------|------|

## Valid merges

| Branch | Merge commit | CI run | Status | Date |
|--------|-------------|--------|--------|------|

## Potential conflicts

> If you edit a file already listed in another worktree's "In progress", note it here.

| File | Affected worktrees | Risk | Resolution |
|------|--------------------|------|------------|

## Decisions made

> Technical choices made during the sprint that affect other sessions.
> Workers: read this section BEFORE coding, a choice made by another worker may change your approach.

| Decision | Reason | Impacts | By | Date |
|----------|--------|---------|----|------|

## Quality Gates — Metrics (filled by the boss)

> The boss fills this section after every audit. Workers: if a gate fails, fix before resubmitting.

| Gate | Scope | Score | Threshold | Status | Date |
|------|-------|-------|-----------|--------|------|
{quality_gates_rows}

### Strong couplings (from /cli-audit-tangle)

> Boss fills this before assigning tasks. Workers: do NOT touch coupled files assigned to another worker.

| Module/Function | Coupling | Assigned to |
|------------------|----------|-------------|

## Sensitive zones (3/3 unanimity quorum)

> Reviewed at the end of every sprint. If a file causes problems, move it here.
> If a file stays here for 3 sprints with no incident, demote it back to the normal zone.
>
> **Machine-parseable format**: the authoritative list is the `BOSS_SENSITIVE_PATHS`
> block below. The human table is generated from that block and serves as documentation.
> The Sous-Chef `/loop` parses only the block — always keep it up to date.

<!-- BOSS_SENSITIVE_PATHS:START -->
```sensitive-paths
# One glob pattern per line. Lines starting with # are comments.
# Parsed by /loop Sous-Chef and the Chef shutdown protocol.
# Format: <glob>    # <reason>
.github/workflows/**          # CI = global impact
**/Cargo.toml                 # Supply chain risk (Rust deps)
**/package.json               # Supply chain risk (npm deps)
**/go.mod                     # Supply chain risk (Go deps)
**/pyproject.toml             # Supply chain risk (Python deps)
**/requirements*.txt          # Supply chain risk (Python deps)
.env                          # Secrets
**/*.secret                   # Secrets
**/credentials*               # Secrets
src/auth/**                   # Critical authentication
src/security/**               # Critical security
CONTRACTS.md                  # Project rules
CONTRIBUTING.md               # Project instructions
```
<!-- BOSS_SENSITIVE_PATHS:END -->

### Human table (generated from the block above)

| Pattern | Reason | Since | Incidents |
|---------|--------|-------|-----------|
| .github/workflows/** | CI = global impact | initial | - |
| **/Cargo.toml | Supply chain risk (Rust) | initial | - |
| **/package.json | Supply chain risk (npm) | initial | - |
| .env, **/*.secret, **/credentials* | Secrets | initial | - |
| src/auth/** | Critical authentication module | initial | - |
| src/security/** | Critical security module | initial | - |
| CONTRACTS.md, CONTRIBUTING.md | Project rules | initial | - |

### How to parse the block (for /loop or scripts)

```bash
# Extract pure patterns (no comments, no markers)
sed -n '/<!-- BOSS_SENSITIVE_PATHS:START -->/,/<!-- BOSS_SENSITIVE_PATHS:END -->/p' \
  {project}/.claude/shared-state.md \
  | sed -n '/```sensitive-paths/,/```/p' \
  | grep -v '^```' \
  | grep -v '^#' \
  | grep -v '^$' \
  | awk '{print $1}'
```

### Hallucination history

> If a Sous-Chef APPROVEd a diff that caused a problem, record it here.
> Used to improve the Sous-Chefs' rules in the next sprint.

| Sprint | Sous-Chef | File | What happened | Fix applied |
|--------|-----------|------|----------------|-------------|

## Shared context

> Information every worker must know.

{context_items}
```
