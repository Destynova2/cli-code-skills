# Sprint Persistence — Checkpoint, Resume, Rewind

> **When to read:** When the user wants to pause/resume a sprint, rewind to a previous state, or start fresh while keeping learnings.

---

## Concept (inspired by the jj operation log)

Every sprint leaves **checkpoints** — snapshots of the complete project state at key moments. The Chef can navigate between these checkpoints like `jj undo` or `git reset`.

```
Sprint S1          Sprint S2          Sprint S3
  │                  │                  │
  ├─ checkpoint-0    ├─ checkpoint-0    ├─ checkpoint-0
  │  (before sprint) │  (before sprint) │  (before sprint)
  │                  │                  │
  ├─ checkpoint-1    ├─ checkpoint-1    │  ← you are here
  │  (after Wave 1)  │  (after 2 plats) │
  │                  │                  │
  ├─ checkpoint-2    ├─ checkpoint-2    │
  │  (after Wave 2)  │  (end of sprint) │
  │                  │                  │
  └─ report-S1       └─ report-S2       │
```

The user can:
- **Pause**: stop the sprint, save state, resume later
- **Resume**: relaunch from the last checkpoint
- **Rewind**: go back to a previous checkpoint (like `jj undo`)
- **Fresh**: start from scratch but keep the learned gotchas and anti-patterns

---

## Checkpoints (automatic saves)

### When to save a checkpoint

| Event | Git tag | Saved content |
|-------|---------|---------------|
| Sprint start (Phase 0 complete) | `sprint/{id}/start` | shared-state.md, PERT, coupling matrix |
| After every merged plat | `sprint/{id}/plat-{n}` | shared-state.md, cumulative diff, scores |
| Before shutdown | `sprint/{id}/pre-shutdown` | full state before the report |
| End of sprint | `sprint/{id}/done` | final report, backlog, sensitive zones |
| User pause | `sprint/{id}/pause` | state at the pause moment |

### How to save

```bash
# The Sous-Chef runs after each merged plat:
git tag "sprint/${SPRINT_ID}/plat-${PLAT_NUMBER}" HEAD
cp .claude/shared-state.md .claude/sprint-history/${SPRINT_ID}/checkpoint-${PLAT_NUMBER}.md
echo "$(date -Is) checkpoint-${PLAT_NUMBER} score=${SCORE}" >> .claude/sprint-history/${SPRINT_ID}/log.txt
```

### On-disk layout

```
{project}/.claude/sprint-history/
├── S1/
│   ├── checkpoint-0.md     (shared-state at the start)
│   ├── checkpoint-1.md     (after plat 1)
│   ├── checkpoint-2.md     (after plat 2)
│   ├── report.md           (final report)
│   ├── log.txt             (timeline: timestamps, scores, events)
│   └── gotchas-learned.md  (anti-patterns discovered during S1)
├── S2/
│   ├── ...
└── current -> S3           (symlink to the sprint in progress)
```

---

## Pause (clean stop)

The user types `pause`, `Ctrl+C`, or closes tmux.

The Chef detects the stop and saves:

```
1. Git tag: sprint/{id}/pause
2. Copy shared-state.md → sprint-history/{id}/checkpoint-pause.md
3. Log: "PAUSED at plat {n}/{total}, score {x}/10"
4. Write to shared-state.md "Status" section: "PAUSED"
5. List the remaining tasks in the "Backlog" section
```

**tmuxinator can be stopped cleanly:** `tmuxinator stop {session}`

---

## Resume (pick up where you left off)

The user re-runs `/cli-forge-boss` on the same project.

The Chef detects a paused sprint:

```
1. Read .claude/sprint-history/current/checkpoint-pause.md
2. Read the log.txt to learn the progress
3. Rebuild the PERT with only the remaining plats
4. Check git state: are the commis branches still there?
   - YES → reassign commis to their existing branches
   - NO → recreate the worktrees
5. Resume the sprint at the next plat
```

**Presentation to the user:**
```
Sprint S3 detected in pause.
  Plats done: 2/5 (auth-refactor, api-endpoints)
  Plats remaining: 3 (db-migration, tests-e2e, docs-update)
  Current score: 7.2/10
  Last activity: 3h ago

  [1] Resume (3 plats remaining)
  [2] Rewind to the previous checkpoint
  [3] Start fresh (keep the gotchas)
  [4] Abandon the sprint
```

---

## Rewind (go back)

Like `jj undo` — return to a previous state.

```
1. List the available checkpoints:
   sprint/S3/start          (4h ago, score 6.8)
   sprint/S3/plat-1         (3h ago, score 7.1) ← auth-refactor
   sprint/S3/plat-2         (2h ago, score 7.2) ← api-endpoints
   sprint/S3/pause          (1h ago, score 7.2)

2. The user picks a checkpoint

3. Restore:
   git reset --hard sprint/S3/plat-1
   cp .claude/sprint-history/S3/checkpoint-1.md .claude/shared-state.md
   # Plats 2+ are cancelled — the commis branches still exist
   # but their merges are reverted

4. Resume from that point (tasks 2+ are re-added to the PERT)
```

**Use cases:**
- A merged plat broke CI → rewind before that plat, reassign to another commis
- The Chef made a bad split → rewind to the start, replan

---

## Fresh restart (start from zero)

When the sprint is too broken to repair.

```
1. Save the learnings:
   - Copy gotchas-learned.md to the next sprint
   - Copy the sensitive-zone list (updated during the sprint)
   - Copy the detected anti-patterns (B1-B10)

2. Reset:
   git reset --hard sprint/S3/start  (or main)
   Delete the commis branches
   Delete the worktrees

3. Launch a new sprint S4:
   - The Chef reads gotchas-learned from S3
   - The sensitive zones are already up to date
   - The PERT is rebuilt from scratch
```

---

## Sprint history (navigation)

The Chef can browse the sprint history:

```
/cli-forge-boss --history

Sprint History — {project}

| Sprint | Date | Plats | Score | Status | Duration |
|--------|------|-------|-------|--------|----------|
| S1 | 2026-03-15 | 3/3 | 7.5 | DONE | 45min |
| S2 | 2026-03-22 | 4/5 | 8.1 | DONE | 1h20 |
| S3 | 2026-04-07 | 2/5 | 7.2 | PAUSED | 2h |

Checkpoints S3: start, plat-1, plat-2, pause
Gotchas learned: 2 (Hot File Thrashing on ci.yml, Ghost Commis timeout)
```

---

## Integration with jj and git

The system works with both:

| Operation | git | jj |
|-----------|-----|-----|
| Checkpoint | `git tag sprint/{id}/...` | `jj bookmark set sprint-{id}-...` |
| Rewind | `git reset --hard {tag}` | `jj undo --to {operation}` |
| Resume | Read tag + restore shared-state | `jj restore --from {bookmark}` |
| Commis branches | `git worktree add` | `jj workspace add` |
| History | `git tag -l "sprint/*"` | `jj op log` |

jj is native for this kind of operation (operation log, undo, workspace). git requires manual tags and worktrees.

---

## Safety rules

1. **Never force-push** the checkpoint tags — they are immutable
2. **Rewind = local reset only** — does not touch the remote until the user approves
3. **Fresh restart keeps the learnings** — gotchas and sensitive zones survive
4. **Automatic checkpoints = no opt-in** — saved after every merged plat, always
5. **Max 10 sprints in history** — older ones are archived (tag stays, files removed)
