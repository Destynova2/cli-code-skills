# Output Template — OCI Rootless Migration Assessment

Produce this structure unless the user explicitly asks otherwise.

```markdown
# OCI rootless migration assessment

## 1. Executive summary
- What the historical system really does.
- Current migration state.
- What is already refined/converged.
- What remains brittle or unproven.

## 2. Bedrock operational contract
### Instance
### Storage
### Identity and privileges
### Security
### Day-2 operations
### Observability
### Recovery

## 3. Strata and tailings not to copy
| Artifact | Encoded behavior | Classification | Reason | Action |

## 4. Fault-line map
| Fault line | Current evidence | Target decision | Required proof |

## 5. Target OCI alloy
### Runtime image
### Init image or init logic
### Tools image
### Check image
### Host bootstrap
### Quadlet and systemd --user
### Public operator CLI
### Compatibility layer

## 6. Migration plan by phases
1. Survey
2. Assay
3. Boundary design
4. Alloy design
5. Prototype
6. Heat treatment
7. Field acceptance
8. Tailings remediation
9. Stratigraphic memory

## 7. Test and proof matrix
| Level | Gate | Evidence now | Required proof | Automation target | Owner |

## 8. Maturity scorecard
| Dimension | Score (0-4) | Evidence | Gap |

(See scoring dimensions below)

## 9. Caveats and residual debt
- Works but insufficiently proven.
- Documented but not automated.
- Still dependent on legacy strata.
- Fault lines requiring production-like testing.

## 10. Decision
**Converged / Partially converged / Non converged**

Reason:

### Recommended handoffs
| Skill | Trigger | Input scope |
```

## Scoring dimensions (0-4 each, target >= 45/60 or 75%)

| # | Dimension | 0 | 2 | 4 |
|---|-----------|---|---|---|
| D1 | Bedrock clarity | No contract extracted | Partial, some unknowns | All 7 sub-contracts explicit with evidence |
| D2 | Strata classification | Artifacts unclassified | Most classified, some Unknown | All classified Preserve/Refine/Adapter/Tailings |
| D3 | Rootless boundary | Fault lines unmapped | Key faults identified | All faults mapped with phase conditions |
| D4 | Image separation | Monolithic or missing | Runtime + tools exist | Runtime/init/tools/check each with one owner |
| D5 | Image hardening | No multi-stage, root user | Multi-stage, non-root | Full: read-only, no pkg mgr, no shell, HEALTHCHECK, CVE scan, SBOM |
| D6 | Host bootstrap | Manual or undocumented | Scripted but partial | Idempotent, covers user/dirs/subuid/certs/Quadlet |
| D7 | CLI consolidation | Scattered scripts | Some commands in CLI | One CLI routes all day-2 ops, wrappers are adapters only |
| D8 | T0 contract proof | No validation | Syntax checks only | Contract inventory + drift guards + manifest consistency |
| D9 | T1 build proof | Images build | Build + smoke | Build + goss/CST + CVE scan + SBOM |
| D10 | T2 deploy proof | Manual deploy works | Scripted clean deploy | Clean deploy + rerun + reboot survival |
| D11 | T3 operations proof | Start/stop only | Day-2 partially proven | Full: monitoring, backup/verify/restore, rotation, diagnostics |
| D12 | T4 stress proof | No failure testing | Some negative tests | Missing secret, bad port, disk full, bus down, SELinux denial |
| D13 | M0 memory | No runbook | Runbook exists | Runbook deltas + blackbox entries + anti-regression tests |
| D14 | Monitoring bridge | No monitoring | Local probes only | Local + remote + failure exit codes + degraded detection |
| D15 | Compatibility debt | Second control plane exists | Wrappers exist, untested | Wrappers tested, bounded, sunset planned |
