# Leaderless Execute — Eliminating the Chef bottleneck via pré-signed tickets

## The problem (observed on grob-s5, 2026-04-17)

In Brigade, every commis permission request flows through the Chef:

```
commis-X  →  permission request  →  Chef (team leader)  →  approve/deny
```

When the Chef thinking-loops for 30+ min, the entire team stalls. Observed: 4 of 6 commis never spawned, 2 of 7 PRs merged, sprint ran 1h30min before manual killing.

## The solution — tickets pré-signés + contre-chef-inter routing

At **Phase 0 (Reflect)**, the Orchestrator signs a **ticket** per commis. The ticket declares the scope the commis is pre-authorized to modify. During **Phase 1 (Execute)**, edits are validated against the ticket by the contre-chef-inter — **no Orchestrator round-trip**.

## Ticket schema

```json
{
  "ticket_id": "t-b05-commis-1-sprint-grob-s5",
  "commis": "commis-1",
  "sprint": "grob-s5",
  "scope": {
    "allowed_files": [
      "src/server/helpers.rs",
      "src/server/helpers_test.rs",
      "tests/integration/routing.rs"
    ],
    "allowed_globs": [
      "shared-state.md",
      "shared-state.jsonl"
    ],
    "forbidden_globs": [
      "**/Cargo.toml",
      "**/*.lock",
      ".github/**",
      "src/auth/**"
    ]
  },
  "budget": {
    "max_edits": 50,
    "max_diff_lines": 500,
    "timeout_minutes": 180
  },
  "issued_at": "2026-04-17T10:00:00Z",
  "expires_at": "2026-04-17T13:00:00Z",
  "signatures": [
    {"signer": "orchestrator", "sig": "..."},
    {"signer": "plan-scope", "sig": "..."},
    {"signer": "plan-secu", "sig": "..."}
  ]
}
```

### Why signatures matter
A ticket is a **capability**, not a request. The contre-chef-inter trusts the signature; it does not re-ask the Orchestrator. If the sig verifies against the issuing quorum's public keys, the scope is authoritative.

### Why scope is explicit
Brigade's failure mode: a commis edits `src/server/helpers.rs` (its scope) **and also** `.github/workflows/` (outside scope) without raising an explicit escalation. A ticket with `forbidden_globs` makes the escalation unmissable.

## Contre-chef-inter: ticket verifier, not approver

The contre-chef-inter in REC-Quorum has a narrower role than in Brigade:

```
Every 10 seconds:
  1. Capture Chef pane for pending inter-agent permission cards
  2. For each pending card:
     a. Extract (commis, tool, file) from the card
     b. Load the commis's ticket (cached in memory)
     c. Verify signature chain (orchestrator + plan-validators)
     d. Match file against allowed_files / allowed_globs / forbidden_globs
     e. Check budget (max_edits counter, elapsed time)
     f. If IN SCOPE and within BUDGET → auto-approve:
        SendMessage to Orchestrator: "ticket-$ID covers this; proceed"
        Orchestrator's only job: forward the approve signal.
        (Orchestrator is now a rubber-stamp for in-scope edits — O(1) work.)
     g. If OUTSIDE SCOPE or BUDGET EXHAUSTED → escalate to Phase 2 mini-quorum:
        Send to control-scope and control-secu: "commis-X wants off-scope Edit on {file}, vote?"
        2-of-2 APPROVE → temporary scope extension, logged
        Else → DENY with message to commis
```

### Why not bypass the Orchestrator entirely?
Because the Agent Teams protocol in Claude Code routes permissions to the team leader by design. We can't change that from outside. What we **can** do: make the leader's decision trivial (just a signature forward) so stalls don't matter.

Alternative: `mode: bypassPermissions` on commis Agent spawn should theoretically skip the permission request entirely. In practice (grob-s5), this didn't cover all edit scenarios. Ticket + contre-chef-inter is the belt-and-suspenders.

## Budget enforcement

Tickets have budgets. Without them, a commis can edit 10,000 lines via a regex Find/Replace and nobody notices until Control.

```
max_edits           : number of Edit/Write tool calls total
max_diff_lines      : total added/removed lines across all commits
timeout_minutes     : wall-clock budget after which ticket expires
```

When 80% of any budget consumed → contre-chef-inter sends early warning to Sous-Chef-merge ("commis-X at 80% budget, consider merging soon").

When 100% of any budget consumed → ticket is revoked; any further Edit fails to verify. Commis must SendMessage for a budget extension (goes through Phase 2 mini-quorum).

## Append-only log of ticket events

Every ticket action is logged:

```jsonl
{"ts":"...","event":"ticket_issued","id":"t-b05-...","commis":"commis-1","scope":{...},"sig_set":["orchestrator","plan-scope","plan-secu"]}
{"ts":"...","event":"edit_approved","ticket":"t-b05-...","file":"src/server/helpers.rs","edits_used":3,"edits_remaining":47}
{"ts":"...","event":"escalation","ticket":"t-b05-...","requested_file":"Cargo.toml","reason":"forbidden_glob","routed_to":"phase2-mini-quorum"}
{"ts":"...","event":"ticket_expired","id":"t-b05-...","reason":"timeout_minutes","used":{"edits":47,"diff_lines":312}}
```

Rebuild state = fold over the log. Never edit in place.

## How it changes the Chef prompt

The Orchestrator (ex-Chef) prompt becomes much lighter:

**Before (Brigade)**:
```
For every permission request from any commis:
  - Read the edit context
  - Classify zone
  - Check shared-state sensitive list
  - Approve or deny via SendMessage
  - Update shared-state
```

**After (REC-Quorum)**:
```
Phase 0:
  Draft plan, hash, wait for plan-validators quorum.
  Issue tickets (one per commis).
  Disappear.

On incoming permission:
  If ticket-sig-valid: rubber-stamp APPROVE.
  If escalation: route to Phase 2 mini-quorum, do not decide.
```

The Orchestrator does **no classification, no thinking, no free-text reasoning** during Execute. It can thinking-loop all it wants — tickets keep working.

## Cost of tickets

A ticket is issued once at Phase 0. Signing = 3 LLM calls (Orchestrator + 2 plan-validators) per commis × N commis. For a 6-commis sprint: 18 LLM calls, total ~$0.50 with Opus, ~$0.05 with Haiku.

Compare to a single 45-min Chef stall: ~5 Opus sessions × $2 = $10 wasted + 45 min human time. Tickets are a bargain.

## Anti-patterns

- **Ticket without signature**: anyone could forge one. Must be cryptographically signed.
- **Ticket covering `**/*`**: defeats the purpose. Scopes must be narrow enough to be meaningful.
- **No budget**: commis can runaway. Always set max_edits and timeout.
- **Forgetting forbidden_globs**: explicit deny is cheaper than discovering via Phase 2 that a commis modified Cargo.toml.
- **Orchestrator still reasoning during Execute**: the whole point is he doesn't. If the generated prompt has "classify zone", you're back to Brigade — redo.
