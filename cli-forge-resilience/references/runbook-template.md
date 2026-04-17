# Runbook and blackbox templates

> **When to read:** Workflow Steps 6 and 7.

## Suggested file layout

- `docs/runbooks/{service}-runbook.md`
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

## 2. Invariants
| Invariant | Source of Truth | Verification Command | Blast Radius |

## 3. Capture before correction
| Surface | Evidence to save | Why |

## 4. Fast triage
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

## 7. Anti-regression reruns
### Minimum reruns
### Broader matrix reruns

## 8. Closure criteria
- symptom captured
- first useful signal captured
- root cause localized
- source of truth corrected
- durable guardrail added
- relevant levels rerun
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
