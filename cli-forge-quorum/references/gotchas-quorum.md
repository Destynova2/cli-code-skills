# Gotchas — REC-Quorum orchestration

REC-Q inherits all of Brigade's pitfalls (see `cli-forge-chef/references/gotchas-chef.md`, G1–G38) plus these REC-Q-specific ones. Read this file before `Phase 4 — Hard gates`.

**Inheritance map.** Most Brigade gotchas apply directly:

| From `gotchas-chef.md` | Relevance to REC-Q |
|---|---|
| G1 permission UI blocks the leader | The Orchestrator is the team leader — same fix: `--permission-mode bypassPermissions`. |
| G2 use `--append-system-prompt-file` | Same. The 5-10 Orchestrator + validator prompts are long Markdown — never load via `$(cat ...)`. |
| G4 `--teammate-mode tmux` | Validators are easier to monitor with split panes. |
| G24 ccheck window mandatory | Still needed for keyboard permission cards. |
| G29 contre-chef-inter mandatory tier ≥ M | Reused verbatim; REC-Q narrows its role to ticket verification (see `leaderless-execute.md`). |
| G31 kick-off positional arg | Every `claude` pane needs it — even when the system prompt says "start immediately". |
| G32 Agent Teams teammates inherit leader's permissions | **Persists in REC-Q**. Tickets fix the *approval stall*, not the *permission scope*. See Q4 below. |
| G33 apply pane is a standalone pane, not a teammate | Same — the Phase 2 apply quorum (Q5) only makes sense if the apply executor has its own permissions. |
| G34 shell-mangling of prompt via `$(cat ...)` | Same fix: `--append-system-prompt-file`. |
| G35 `git worktree add \| head -1` SIGPIPE | Same fix: reuse `brigade-setup-worktrees.sh` from chef. |
| G36 one branch per commis | Same — applies to per-validator worktrees too if you choose to isolate them. |
| G37 first-time trust prompt | Same hack: `Down Enter` on each claude pane 3s after launch. |

## REC-Q-specific gotchas

## Q1 — Hash canonicalization is not "pretty-printed JSON"

**Symptom:** Plan-Validators vote on the same plan, get different hashes, Phase 0 retries 3× and escalates to human even though both validators would have agreed on the content.

**Root cause:** `JSON.stringify(plan)` produces different output depending on key order, whitespace, number formatting (`1.0` vs `1`), and escape choices (`é` vs `é`). Two validators running the same plan through non-identical serializers produce non-identical hashes.

**Fix:** use a **canonical JSON** serializer — RFC 8785 (JCS) or equivalent. In practice for a generated orchestrator:

```python
import json, hashlib
def canonical_hash(obj):
    # sort_keys=True + no whitespace + ensure_ascii=False + specific float handling
    s = json.dumps(obj, sort_keys=True, separators=(',', ':'), ensure_ascii=False)
    return hashlib.sha256(s.encode('utf-8')).hexdigest()
```

Every agent must use the SAME canonicalization. Embed the code, do not re-implement per-agent.

**Detection:** run the canonicalizer on `{"a":1,"b":2}` and `{"b":2,"a":1}` — hashes must match.

## Q2 — View-change racing with a recovering agent

**Symptom:** An agent stalls at T=9min. Backup spawns at T=10min (timeout). Original agent recovers at T=10min30s and also emits its decision. Now two agents in the same slot, conflicting signatures.

**Root cause:** no lease / fencing mechanism. View-change "replaces" an agent but doesn't revoke its authority.

**Fix:** every agent operates under a **lease token** (monotonic counter). When spawning a backup, increment the lease counter. Signatures include the lease number. The log-appender rejects signatures with stale leases:

```jsonl
{"ts":"T=10min","event":"view_change","old_agent":"plan-validator-2","new_agent":"plan-validator-2-v2","lease_old":1,"lease_new":2}
{"ts":"T=10min30s","event":"reject","actor":"plan-validator-2","reason":"stale_lease (1 < 2)"}
```

The recovering agent's late signature is discarded. Safe.

**Detection:** simulation test — deliberately delay an agent past its timeout then let it emit. Verify its decision is rejected with reason `stale_lease`.

## Q3 — Append-only log is not atomic across agents

**Symptom:** Two commis write to `shared-state.jsonl` simultaneously, their lines interleave mid-line, log becomes unparseable.

**Root cause:** `echo "..." >> file` is not atomic when the line exceeds PIPE_BUF (4096 bytes on Linux). A commis writing a 2KB log entry + another commis writing 3KB = partial lines.

**Fix:** three options, pick one:
1. **Advisory lock**: `flock -x shared-state.jsonl.lock` around the append.
2. **One file per actor**: `{actor}-{timestamp}.jsonl` in a `log.d/` dir, merged by the Orchestrator at phase boundaries (easier for concurrent writes).
3. **Length-prefixed framing**: each entry prefixed with its byte count; parsers handle partial lines.

The chef skill's `brigade-setup-worktrees.sh` does not address this — you need to add the lock or split-file pattern in the Orchestrator prompt. REC-Q's `shared-state.jsonl` bootstrap should include the chosen strategy.

**Detection:** multi-agent smoke test with 5+ agents writing to the log simultaneously. Run `jq -c . shared-state.jsonl > /dev/null` — if it fails with parse errors, atomicity is broken.

## Q4 — Ticket scope ≠ settings.local.json permissions

**Symptom:** A commis has a ticket that says `allowed_files: ["src/server/**"]` but its Bash tool call `rm -rf src/` succeeds because the root `settings.local.json` doesn't deny it.

**Root cause:** tickets are a *logical* capability check done by `contre-chef-inter` on Edit/Write/Read tool calls. They do NOT constrain Bash. A commis inheriting the Orchestrator's `settings.local.json` has whatever Bash permissions the Orchestrator has.

**Fix:** two-layer defence:
1. Root `settings.local.json` (inherited by all teammates per G32) denies destructive Bash primitives: `Bash(rm -rf:*)`, `Bash(git push --force*:*)`, `Bash(tofu apply:*)`, `Bash(helm upgrade:*)`, `Bash(kubectl apply:*)`, etc.
2. Tickets constrain Edit/Write to specific paths. These layer on top of (1), they don't replace it.

**Detection:** `grep -c 'Bash(rm -rf' {project}/.claude/settings.local.json` → must be ≥ 1 in the deny list. Same for `tofu apply`, `helm upgrade`, `kubectl apply`. If missing, tickets alone won't stop a rogue commis.

## Q5 — Apply commands belong in Phase 2, not Phase 1

**Symptom:** A commis whose scope includes `terraform/*.tf` runs `tofu apply` during Phase 1 (Execute). The ticket lets it Edit the file, the Bash allow-list lets it apply (if misconfigured). Infra changes without validation.

**Root cause:** Phase 1 is for **producing** artifacts; Phase 2 is for **validating** them. Running an apply during Phase 1 fuses the two. `tofu plan` is the producer output, `tofu apply` should happen after Phase 2 approves the plan.

**Fix:** strictly separate `plan` (Phase 1 allowed, in-scope commis only) from `apply` (Phase 2 only, via the apply pane after 3/3 Control-Validator APPROVE). See `apply-quorum-rec-q.md` for the flow. The commis's ticket should have `forbidden_globs: ["**/*.tfstate"]` and the root `settings.local.json` should deny `Bash(tofu apply:*)` / `Bash(helm upgrade:*)` / `Bash(kubectl apply:*)` for the commis world. Only the apply pane (standalone, not a teammate) has those allowed.

**Detection:** grep the generated Orchestrator prompt for any mention of `tofu apply` / `helm upgrade` / `kubectl apply` in Phase 1. These should only appear in Phase 2 context or in the apply pane's prompt.

## Q6 — Phase 2 validators cloning each other

**Symptom:** All 5 Control-Validators use the same prompt template with only `{role}` substituted. They all produce identical APPROVE votes even when there's a real bug.

**Root cause:** BFT assumes **independent** faults. Validators with the same prompt + same model have perfectly correlated failures — a blind spot in one = blind spot in all. You get N=5 in name only.

**Fix:** each validator gets a **distinct focus**:
- `control-tests`: runs tests, only cares about green/red
- `control-security`: reads diff with a secu lens, checks for secrets, IAM widening, new ingress
- `control-contracts`: checks API surface against `CONTRACTS.md`
- `control-policy`: enforces policy-as-code (OPA, Conftest, etc.)
- `control-perf`: runs benchmarks, flags regression

And ideally **different model families** (Opus + Sonnet + Haiku mix) so even silent model-specific blind spots differ.

**Detection:** diff the generated validator prompts. If two are literally identical up to `{role}`, you're not getting real BFT.

## Q7 — Orchestrator prompt still contains classification logic

**Symptom:** The generated Orchestrator prompt contains paragraphs like "For each commis permission request, classify zone (normal / sensitive) and decide APPROVE or DENY". This is Brigade's Chef behaviour. REC-Q forbids it.

**Root cause:** the generator copy-pasted Brigade's prompt and didn't strip the decision paragraphs. The single-leader bottleneck returns.

**Fix:** during Execute, the Orchestrator's prompt section must say LITERALLY:
```
Phase 1: You do not classify. You do not decide. You forward `contre-chef-inter`'s
recommendation unchanged. If `contre-chef-inter` says APPROVE, you SendMessage
APPROVE to the requester. If ESCALATE, you route to Phase 2 mini-quorum.
You do not read the diff. You do not read the file. You forward.
```

If the prompt contains "classify", "check the zone", "look at the file", "decide whether", the generator is back to Brigade. Regenerate.

**Detection:** `grep -iE "classify|decide|zone" orchestrator-{session}.md` → should match 0 times in the Phase 1 section.

## Q8 — Certificates stored in-prompt, not on disk

**Symptom:** The Phase 0 certificate is "stored" by telling the Orchestrator to remember the plan hash in its context. After a restart or view-change, it's gone.

**Root cause:** LLM context is ephemeral. Agent Teams **loses context on view-change** (new claude process = new context).

**Fix:** certificates are written to `{project}/.claude/rec-quorum/certificates/{phase}-{session}-{timestamp}.json` the moment they're emitted. The file is signed and the signature is the authoritative record. The Orchestrator reads from disk at Phase transitions, never from memory.

**Detection:** `ls {project}/.claude/rec-quorum/certificates/` after a completed sprint. Must contain one file per phase transition.

## Q9 — Tickets signed once, never rotated

**Symptom:** A ticket issued at T=0 with `expires_at: T+3h` is used at T=2h58min. The commis's Edit succeeds. At T=3h, same Edit repeated (maybe a retry) — now silently fails with "ticket expired". Commis gives up.

**Root cause:** short-horizon expiry + no auto-refresh. A normal commis should be able to refresh its ticket near expiry if it's making progress.

**Fix:** tickets include a `refreshable_until` (say, T+5h) — past `expires_at` but before `refreshable_until`, the commis can SendMessage the Plan-Validators (2/2) for a refresh certificate. This is cheap (~$0.10) and avoids orphaned work.

After `refreshable_until`, hard stop — the ticket is dead, the sprint may need re-planning.

**Detection:** simulate a commis reaching 95% budget consumption. Its ticket should either (a) trigger a refresh, or (b) cleanly abandon and report to the Sous-Chef-merge. Silent failure = bug.

## Q10 — Petri net "looks OK" but soundness check skipped

**Symptom:** the generated `.bpmn` / `.cpn` visualizes correctly in WoPeD, but the soundness check (reachable-from-start, no dead transitions, bounded tokens) was not run. At Phase 1, a transition is never fired because its guard is unreachable — sprint silently stalls.

**Root cause:** visual diagrams look plausible even when the underlying net has unreachable states or dead transitions. WoPeD / Renew can PROVE these properties with `Soundness Check`, but the tool must actually be run.

**Fix:** the `Phase 4 hard gate` must invoke the soundness checker programmatically and exit non-zero on failure. Example with WoPeD-CLI:

```bash
woped-cli --soundness-check {project}/.claude/rec-quorum/petri-net.cpn || exit 1
```

Or use a Python library like `snakes` to check reachability in a small unit test.

**Detection:** run the check. If it's not part of `Phase 4 hard gate`, it's not being run in practice.
