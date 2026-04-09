# Quality Gates — CLI skills for merge gates

## Available gates

| Skill | When | Blocks if | Token cost |
|-------|------|-----------|------------|
| `/cli-audit-tangle` | Pre-cycle (Phase 0) | God functions, cycles | High |
| `/cli-audit-drift` | Per merge | Misses (code ≠ CONTRACTS.md contract) | Medium |
| `/cli-audit-code` | Per merge | CQI drops by > 0.2 | Medium |
| `/cli-audit-test` | Per merge | Test pyramid unbalanced | Medium |
| `/cli-audit-sync` | Post-merge (if docs) | Docs out of sync | Medium |
| `/cli-cycle` | End of cycle | Global regression | High |

## When to use what

### Minimum viable (any project)

```
Phase 0: /cli-audit-tangle -> assign modules
Per merge: /cli-audit-code -> minimum quality
End of cycle: /cli-cycle -> scorecard
```

### Standard (project with contracts/docs)

```
Phase 0: /cli-audit-tangle
Per merge: /cli-audit-code + /cli-audit-drift
Post-merge docs: /cli-audit-sync
End of cycle: /cli-cycle
```

### Full (enterprise/regulated project)

```
Phase 0: /cli-audit-tangle
Per merge: /cli-audit-code + /cli-audit-drift + /cli-audit-test
Post-merge docs: /cli-audit-sync
End of cycle: /cli-cycle
```

## Integration in the boss prompt

```markdown
## Quality Gates

### Protocol

1. Worker sends "Task complete" via SendMessage
2. Boss runs the gates in the gate window:
   /cli-audit-code {scope}
   /cli-audit-drift {scope}
3. If a gate fails:
   SendMessage { to: "{worker}", summary: "gate failed", message: "GATE FAILED: {detail}. Fix and recommit." }
4. Wait for the fix, re-run the gate
5. If the gate passes: merge
```

## Protocol when a gate fails

1. Read the audit report
2. Identify the files/functions at fault
3. Send the worker a precise message:
   - Which gate failed
   - Which file/function is at fault
   - What the contract/rule expected
   - What the code does instead
4. Wait for the worker's fix
5. Re-run the gate
6. If 3 consecutive failures: escalate to the user
```
