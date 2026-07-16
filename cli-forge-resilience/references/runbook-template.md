# Runbook and blackbox templates

> **When to read:** Workflow Steps 6 and 7.

## Contents

- Suggested file layout
- Runbook template
- Incident blackbox template
- Recurrent hurdle template
- Writing rules

---

## Suggested file layout

- `docs/runbooks/{service}-runbook.md`
- `docs/operations/{service}-agent-policy.md`
- `docs/operations/{service}-capability-contracts.yaml`
- `docs/operations/{service}-escalation-matrix.md`
- `docs/testing/{service}-resilience-matrix.md`
- `docs/incidents/incident-blackbox.md`
- `docs/incidents/postmortems/{date}-{incident}.md`

---

## Runbook template

```md
# {Service} — Operations Runbook

## 1. System identity
- Purpose:
- Owners:
- Runtime:
- Critical dependencies:
- Primary source(s) of truth:
- Support tier owner:
- Default agent autonomy:
- Last reviewed:

## 2. Invariants
| Invariant | Source of Truth | Verification Command | Blast Radius |

## 3. Capture before correction
| Surface | Evidence to save | Why |

Run the SAME check before and after the fix, to a timestamped file — the diff is the proof:
`<check> > before-$TS.txt 2>&1`, apply the bounded action, then `<check> > after-$TS.txt 2>&1`, then `diff before-$TS.txt after-$TS.txt`.

## 4. Fast triage
_Each step below = one concrete command + its expected output (OK) and failing output (KO), not a bare verb._
1. Verify current revision / version / image ref
2. Check service status
3. Check health
4. Check first useful logs
5. Test operator path
6. Test external client path
7. Check monitoring path
8. Check storage / network / identity path relevant to the symptom

## 5. Decision tree
### A. Fails at startup
### B. Starts but health is bad
### C. Operator path works, external path fails
### D. External path works, monitoring fails
### E. Rerun/rollback fails
### F. Backup/restore or migration path fails

## 6. Recovery / containment
- Safe rollback:
- Safe stop:
- Blast-radius reduction:
- What NOT to delete before evidence capture:

## 7. Agent readiness
| Action | Support Tier | Max Autonomy | Preconditions | Blast Radius | Approval | Escalation |

### Capability contracts
- Contract source:
- Audit trail:
- Human-only boundaries:

## 8. Anti-regression reruns
### Minimum reruns
### Broader matrix reruns

## 9. Closure criteria
- symptom captured
- first useful signal captured
- root cause localized
- source of truth corrected
- durable guardrail added
- relevant levels rerun
- agent policy / capability contracts updated if an agent can execute the operation
- runbook/blackbox updated
```

---

## Incident blackbox template

```md
## INC-0XX — Short title

### Problem observed

### Typical symptoms

### First useful signal

### Source of truth corrected

### Resolution method

### Durable solution retained

### Anti-regression tests to keep

### Prevention

### Evidence
- commits:
- files:
- logs / artifacts:
```

---

## Recurrent hurdle template

```md
## HUR-0XX — Short title

### Problem observed

### Typical confusion pattern

### Disambiguation question
- Ask first: "Which path is failing exactly?"

### Resolution method

### Durable clarification retained

### Anti-regression tests to keep

### Prevention

### Conversation sources
- dates / context:
- impact observed:
```

## Writing rules

1. Start from an observed symptom, not a vague impression.
2. Name the **first useful signal** explicitly.
3. Name the **source of truth corrected** explicitly.
4. Attach at least one durable guardrail.
5. If the same confusion is likely to recur, update both runbook and blackbox.
6. Every diagnostic step gives **both** the concrete, copy-pasteable command (explicit `<placeholders>` — never a vague verb like "check the backend") **and** its expected result: what a healthy output (OK) and a failing output (KO) look like. A command without its expected result is half a step.
7. **Never fabricate command output.** When a real capture is required, emit a clearly-marked placeholder (🚧 "replace with real output / capture"), never invented text that looks authoritative.

## Reference grammar (the Stripe rule)

Model the runbook's mechanics on a great API reference (Stripe's is the benchmark: example first, fixed per-resource skeleton, `Returns X…. Raises an error if…`). Seven mechanisms transfer directly:

1. **Fixed skeleton per repeated unit.** Every procedure, every decision branch, every incident entry uses the identical block order (symptom → numbered commands → escalation, or the template above). The reader learns the map once; a unit that deviates costs a re-scan. Stripe orders every resource the same way (object → create → update → retrieve → list → delete) for this reason.
2. **Concrete before abstract.** Inside each unit, the copy-pasteable command or real example is the first block; prose qualifies it afterwards. Stripe shows the curl call and the JSON object before the parameter dictionary.
3. **Sentence templates.** Reuse the same phrase shapes throughout: one template for OK, one for KO, one for escalation triggers. Success AND failure are always both stated ("Returns…" / "Raises an error if…" — never success alone). Every KO names the next action (a section, a command, or an escalation level) — a failure with no pointer is a dead end.
4. **One verb per operation.** Fix a small verb set (verify, capture, restart, replay, escalate…) and use exactly one verb per meaning across the whole document — no synonym drift (restart vs relaunch vs reboot for the same gesture). Stripe never says "fetch" where it says "retrieve".
5. **Typed, badged entries.** Every variable/parameter row carries its type, default, and required/optional (or conditional) status; every mutating command carries its minimum support tier (N1/N2/N3) — the runbook's equivalent of Stripe's `(string, required)` badges.
6. **Deep links and a stitched spine.** Every cross-reference ("see section X") is a clickable link to a validated anchor, and the operational spine is fully linked: triage routes to every decision branch, every decision branch links to its procedure, every failure points to escalation. Check inbound references — a procedure no other section points to is unreachable mid-incident (found in practice: three procedures with zero inbound links because the decision tree never pointed to them). Beware dual renderers: GitLab and pandoc compute different heading anchors by default (`+gfm_auto_identifiers` aligns them), and a renamed heading breaks its inbound links silently — validate anchors mechanically, ideally as a blocking render gate.
7. **The render is the doc's runtime.** For docs shipped as PDF or site, the render job is the gate: blocking (no `allow_failure`), failing on broken internal anchors and on layout overflow (zero `Overfull \hbox` on the LaTeX log). A lint pass on the source proves nothing about what the operator will actually hold in hand.
