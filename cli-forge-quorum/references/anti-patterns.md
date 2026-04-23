# Anti-Patterns — what NOT to generate

REC-Quorum is powerful, easy to misapply. Refuse to generate any of these.

## 1. Same quorum for all tasks

**Symptom**: the generated orchestrator uses `N = 5` (or any fixed value) regardless of task criticality.

**Why it's wrong**:
- Trivial tasks at N=5 waste 5× cost (docs typo fix shouldn't cost as much as a crypto review)
- Critical tasks at N=5 under-protect if they're actually critical (crypto/payment tolerates only 1 Byzantine, but should tolerate 2)

**Correct**: map each task through the criticality classifier (`quorum-sizing.md`). N adapts per task.

## 2. Majority vote on free text

**Symptom**: Phase 0 Plan-Validators vote "APPROVE / DENY" via SendMessage free-form text, compared by string match.

**Why it's wrong**: two LLMs asked "is this plan good?" answer in different words even when they agree. Non-reproducible. Paper [arxiv 2511.15755](https://arxiv.org/abs/2511.15755) proved zero-variance requires deterministic inputs to voting.

**Correct**: canonicalize the plan (sorted keys, no whitespace variance), hash it (SHA-256), vote on the hash. Two validators either agree on the hash or not. No free text comparison.

## 3. Reflect and Execute fused

**Symptom**: a single agent "plans and executes" in one pass.

**Why it's wrong**: you lose the planning gate. If the plan is wrong, you discover it after execution costs have been spent. Reintroduces Brigade's Chef bottleneck (the one entity that decides *and* does).

**Correct**: Phase 0 is planning ONLY, no tool execution. Phase 1 starts with a signed plan certificate in hand.

## 4. Skip Phase 2 ("ça marche en dev")

**Symptom**: the generated orchestrator merges PRs after Phase 1 with no Phase 2.

**Why it's wrong**: Phase 2 is where the audit trail is built. Compliance requires it. SLA requires it. Without it, you have Brigade, not REC-Quorum.

**Correct**: Phase 2 is mandatory for routine/sensitive/critical tasks. Trivial tasks can skip Phase 2 only if the criticality classifier explicitly set N=1.

## 5. Threshold signatures on every operation

**Symptom**: every Edit/Write tool call triggers a BLS signing round across N validators.

**Why it's wrong**: overhead explodes. Signing is ~10ms of CPU per validator + network round-trip. For a commis doing 50 edits, that's 500 signature rounds × N validators = thousands of signatures per sprint. Cost: massive.

**Correct**: sign only at **phase boundaries** (Phase 0 exit: plan + tickets; Phase 2 exit: control certificate). Intra-phase operations are recorded in the append-only log but not individually consensed.

## 6. Human-in-the-loop as default

**Symptom**: every non-trivial decision escalates to the human operator.

**Why it's wrong**: defeats automation. REC-Quorum should auto-resolve > 98% of cases. If escalations dominate, the classifier is too aggressive or the quorum sizes are too small.

**Correct**: escalation only when Phase 2 Control DENIES or when view-change exhausts (backup after backup stalls). Log everything, page the human only on true escalations.

## 7. Ticket without signature

**Symptom**: the ticket schema has a `scope` but no `signatures` field (or signatures are optional/skipped).

**Why it's wrong**: unsigned tickets are forgeable. A rogue commis could claim arbitrary scope. The whole point of leaderless Execute is that tickets are **capabilities** — unforgeable, verifiable locally.

**Correct**: tickets always include signatures from the Reflect quorum (orchestrator + all plan-validators). Verify chain before acting.

## 8. Ticket covering `**/*`

**Symptom**: a commis ticket has `allowed_globs: ["**/*"]`.

**Why it's wrong**: defeats the ticket system. Might as well not have one. A commis with unlimited scope bypasses all safety checks.

**Correct**: scopes are narrow. Allowed paths are enumerated (or tight globs like `src/server/helpers.rs`, `src/server/helpers_test.rs`). Anything else requires explicit scope extension via Phase 2 mini-quorum.

## 9. No budget on tickets

**Symptom**: tickets have scopes but no `budget` (max_edits, timeout_minutes, max_diff_lines).

**Why it's wrong**: a commis stuck in a regex replace loop can produce 10,000 edits before anyone notices. Budget is the emergency brake.

**Correct**: every ticket has `budget`. When 80% consumed, contre-chef-inter warns. When 100% consumed, ticket expires, further edits fail to verify.

## 10. Orchestrator still reasoning during Execute

**Symptom**: the generated orchestrator prompt says something like "During Phase 1, classify each permission request by zone, check sensitive list, approve or deny."

**Why it's wrong**: this is Brigade's Chef behavior. Reintroduces the single-leader bottleneck REC-Quorum is designed to eliminate.

**Correct**: the orchestrator prompt during Execute says ONLY:
- "If ticket signature verifies: rubber-stamp approve."
- "If outside ticket scope: route to Phase 2 mini-quorum, do not decide."

No classification, no thinking, no free-text. The orchestrator can even thinking-loop — tickets still work.

## 11. Per-phase timeouts missing

**Symptom**: no timeout specified for Phase 0, Phase 1, or individual Phase 2 validators.

**Why it's wrong**: grob-s5 stalled for 45 min. Every phase, every agent, every operation MUST have a timeout. Missing timeout = guaranteed stall eventually.

**Correct**: Phase 0 = 10 min, Phase 1 task = 2× PERT-E, Phase 2 validator = 5 min. Tune per project but never omit.

## 12. Using `shared-state.md` instead of `.jsonl`

**Symptom**: the generated REC-Q bootstrap creates a markdown shared-state file editable in place.

**Why it's wrong**: concurrent writes conflict. No audit trail (git history is not enough if sprints happen multiple times per day). No hash chain integrity.

**Correct**: append-only JSONL with hash chain + signatures. Rebuild state by folding over the log.

## 13. Reinventing BFT

**Symptom**: the generated orchestrator hand-rolls consensus logic ("collect N votes, pick majority, handle Byzantine cases...").

**Why it's wrong**: distributed consensus is subtle. Every hand-rolled BFT has bugs (split votes, livelock, equivocation). Academic protocols exist for a reason.

**Correct**: the orchestrator delegates consensus to an existing library (`HotStuff-lite`, `Mir-BFT`, etc. — see `bft-primitives.md`). If the project doesn't have one installed, generate a stub that panics with install instructions, don't silently run unsigned.

## 14. Petri net skipped because "it's just a diagram"

**Symptom**: the generator outputs prompts but no `.bpmn`/`.cpn`/`.pnml` workflow file.

**Why it's wrong**: the soundness check on the Petri net is the only formal proof that the generated workflow has no deadlock. Skipping it means shipping a process that may hang.

**Correct**: always generate the Petri model. Always run WoPeD/Renew on it before `tmuxinator start`. Fail the hard gate if soundness fails.

## 15. Phase 2 validators identical

**Symptom**: all f+1 Phase 2 validators use the same prompt and same model.

**Why it's wrong**: they'll produce correlated failures. If the prompt has a blind spot (e.g., doesn't check for SQL injection), all N validators miss the same issue. BFT assumes **independent** faults.

**Correct**: each validator has a distinct focus (`control-tests`, `control-security`, `control-contracts`, `control-policy`) AND ideally runs on a different model family (some Opus, some Sonnet, some Haiku). Diversity is the point.

## 16. Auto-approve on timeout

**Symptom**: "If Phase 2 validator doesn't respond in 5 min, assume APPROVE."

**Why it's wrong**: silently turns a fault into an approval. An attacker can deliberately kill validators to coast through Control.

**Correct**: timeout → view-change (spawn replacement validator) OR DENY (conservative default). Never auto-approve on silence.

## 17. Skill refused to print cost estimate

**Symptom**: the generation completes without showing the operator the predicted LLM cost.

**Why it's wrong**: operators pick criticality without understanding the cost. They'll default to `critical` "just to be safe" and burn budget.

**Correct**: always print:
```
Task mix:     T trivial, R routine, S sensitive, C critical
Predicted cost at REC-Q: $Y (about 2-3× Brigade baseline)
Ship anyway? [y/N]
```
Make operators consciously accept the cost.

## 18. Loading the orchestrator prompt via `$(cat ...)` in the tmuxinator YAML

**Symptom**: the generated `~/.config/tmuxinator/{session}.yml` contains `claude --append-system-prompt "$(cat prompts/orchestrator.md)"`. The orchestrator pane dies with bash parse errors or launches with a garbled prompt.

**Why it's wrong**: a realistic orchestrator prompt contains backticks (code samples), `$(...)` snippets (command examples), and unbalanced quotes inside code blocks. `$(cat ...)` expands through the shell before claude sees anything — the shell re-interprets backticks / `$(...)` / quotes in the Markdown. One unbalanced quote in any code sample and bash aborts the command. See `cli-forge-chef/references/gotchas-chef.md` G34 for the full backstory.

**Correct**: use the native `--append-system-prompt-file <path>` flag. The flag reads the file inside claude — the shell sees only the path:

```yaml
- claude --dangerously-skip-permissions --permission-mode bypassPermissions --teammate-mode tmux \
         --append-system-prompt-file /path/to/prompts/orchestrator-{session}.md \
         "Begin Phase 0 per the system prompt."
```

No wrapper script. No `.initial.txt` file. No shell re-parsing.

**Detection**: `! grep -q '\$(cat ' ~/.config/tmuxinator/{session}.yml` must be true.

## 19. Inline `git worktree add ... | head -1` in `on_project_start`

**Symptom**: the tmuxinator `on_project_start` runs, reports success, but only some of the worktrees are actually created. Subsequent re-runs fail because orphan directories block re-creation.

**Why it's wrong**: under `set -o pipefail`, piping `git worktree add` into `head -1` triggers SIGPIPE on git after head closes stdin. Git aborts mid-creation; the worktree registry is left inconsistent. See `cli-forge-chef/references/gotchas-chef.md` G35.

**Correct**: reuse `cli-forge-chef`'s `scripts/brigade-setup-worktrees.sh` which handles idempotency, orphan recovery, and avoids the pipe entirely. Adapt for REC-Q by extending the role list (orchestrator, N plan-validators, N control-validators, apply pane, commis, maître, gate, ccheck, inter). Call it from `on_project_start` with a single line.

**Detection**: `! grep -q 'worktree add.*| head' ~/.config/tmuxinator/{session}.yml`.

## 20. Shared branch across validators or commis

**Symptom**: only the first validator/commis spawns successfully; the rest sit idle because `git worktree add <path> <branch>` silently fails when `<branch>` is already attached elsewhere.

**Why it's wrong**: the same branch cannot be checked out in two worktrees. If the generator assigns `wt/control` to all N Control-Validators, only one succeeds. See `cli-forge-chef/references/gotchas-chef.md` G36.

**Correct**: one branch per agent worktree. Suggested naming for REC-Q:
```
wt/orch-{session}            ← orchestrator
wt/plan-scope                ← plan-validator-scope
wt/plan-secu                 ← plan-validator-secu
wt/control-tests             ← control-validator-tests
wt/control-security          ← control-validator-security
wt/control-contracts         ← control-validator-contracts
{base}-commis-{commis_name}  ← commis i
```

**Detection**: `git branch | grep -c '^  wt/\|^  {base_branch}-'` equals the number of agent panes.

## 21. First-time trust prompt ignored

**Symptom**: the orchestrator + N validator panes sit on the "Bypass Permissions" trust dialog indefinitely. Sprint never starts.

**Why it's wrong**: Claude Code's trust prompt is shown on any new cwd regardless of `--dangerously-skip-permissions`. See `cli-forge-chef/references/gotchas-chef.md` G37.

**Correct**: `on_project_start` includes a delayed `Down Enter` auto-send to every claude pane:

```yaml
on_project_start:
  - (sleep 3 && for w in orchestrator plan-scope plan-secu control-tests control-secu ccheck inter; do
       tmux send-keys -t {session}:$w Down Enter 2>/dev/null || true
     done) &
```

The `&` is mandatory; without it tmuxinator blocks on the sleep.

**Detection**: 30 s after `tmuxinator start`, `tmux capture-pane -p -t {session}:orchestrator | grep -c 'Bypass Permissions'` must be 0.

## 22. Obsolete wrapper script still referenced

**Symptom**: the generated `tmuxinator/{session}.yml` invokes `{project}/scripts/brigade-launch-agent.sh` (or a REC-Q equivalent). The script may or may not exist.

**Why it's wrong**: an earlier generation of `cli-forge-chef` created a wrapper to work around the shell-mangling issue (G34). That wrapper is now obsolete because `--append-system-prompt-file` is natively supported. A new `cli-forge-quorum` generation should not carry the legacy pattern.

**Correct**: never emit `brigade-launch-agent.sh`. If the file exists on disk from a prior iteration, delete it. YAML launches claude directly with the native flag.

**Detection**: `! test -f {project}/scripts/brigade-launch-agent.sh && ! grep -q 'brigade-launch-agent' ~/.config/tmuxinator/{session}.yml`.
