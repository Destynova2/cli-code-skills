# Cross-Skill Handoffs — OCI Rootless Migration

This skill owns migration framing, bedrock extraction, target alloy design, and
convergence verdict. Other skills may be invoked or recommended for specialized
sub-problems. In `--deep` mode, inspect local `cli-*` skill folders before
claiming a handoff exists.

## Handoff table

| Skill | Use when | Input to provide | Expected output |
|---|---|---|---|
| `cli-audit-shell` | Historical scripts, wrappers, backup scripts, checks, cron tasks, unsafe shell mutations | Script list + intended contract + suspected tailings | Shell risk map, refactor candidates, safe adapter guidance |
| `cli-forge-infra` | Host bootstrap, Podman rootless, Quadlet, systemd, storage, network, OS prerequisites | Fault-line map + target runtime assumptions | Concrete infra/runtime design and implementation notes |
| `cli-forge-resilience` | Prod-parity, failure injection, recovery, incident prevention, runbook deltas | T4/M0 gaps + stress scenarios | Resilience gates, degraded-mode plan, runbook deltas |
| `cli-audit-test` | Existing tests need coverage audit | Current CI/test inventory + migration gates | Coverage gaps and test backlog |
| `cli-forge-pipeline` | Need CI/CD implementation for image builds and proof gates | T0/T1/T2/T3/T4 gate matrix + artifact requirements | Pipeline design and executable stages |
| `cli-forge-tree` | Repo structure prevents clean separation of runtime/tools/check/bootstrap | Target alloy map + current tree pain points | Repository layout proposal |
| `cli-forge-schema` | Need schemas for manifests, config, instance descriptors, or generated Quadlet inputs | Bedrock contract + config variables | Schema and validation strategy |
| `cli-forge-hld` | Need high-level architecture doc for stakeholders | Executive summary + target alloy + risks | HLD |
| `cli-forge-lld` | Need implementation-level design for CLI, images, Quadlet, or bootstrap | Fault map + artifact responsibilities + gate requirements | LLD |
| `cli-forge-doc` | Need runbooks, operator docs, migration docs | Public CLI + day-2 contract + caveats | Documentation/runbook package |
| `cli-forge-chef` | User wants multi-agent implementation sprint | Migration backlog + work packages + gates | Brigade-style implementation plan |
| `cli-forge-quorum` | High-stakes, regulated, large-scope governance or consensus needed | Evidence packets + invariants + acceptance criteria | Quorum-reviewed plan/verdict |
| `cli-audit-drift` | Explicit invariants or contracts exist and may drift | Bedrock inventory + generated artifacts | Drift findings and T0 gates |
| `cli-audit-sync` | Docs, code, manifests, generated outputs disagree | Strata map + duplicate truth candidates | Sync/drift report |

## Handoff packet template

```markdown
# Handoff request for {skill}

## Migration context
Historical model:
Target model:
Tier:
Scenario:

## Evidence scope
Paths/files inspected:
Relevant commands:

## Bedrock contract excerpt
- Instance:
- Storage:
- Identity:
- Security:
- Day-2:
- Observability:
- Recovery:

## Fault lines / questions for this skill
1.
2.
3.

## Required output
- Claims with evidence
- Risks
- Gates or implementation recommendations
- Unknowns
```

## Handoff rules

- Do not hand off vague work.
- Do not delegate the final convergence decision.
- When another skill returns implementation detail, translate it back into the
  migration proof matrix.
- Handoffs cannot replace evidence. Missing evidence remains Missing proof.
