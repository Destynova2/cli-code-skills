# Scenario Playbooks — Lithic OCI Rootless Migration

Use these scenario cards to route analysis, sub-agents, handoffs, and output
focus. A repository can match several scenarios.

## Scenario A — Ansible stateful service to rootless product

**Signals:** `roles/`, `tasks/`, `handlers/`, `templates/`, `group_vars/`, root
systemd units, data directories, backup scripts, monitoring checks.

**Primary agents:** Bedrock Surveyor, Stratigraphy Archaeologist, Fault-Line
Geomechanic, Reservoir & Recovery Engineer, Field Operator Ethnographer, Assay
Gate Engineer, Fracture Risk Analyst.

**Handoffs:** `cli-audit-shell`, `cli-forge-infra`, `cli-forge-resilience`,
`cli-audit-test`.

**Key questions:**

- Which Ansible variables are bedrock contract vs installation strata?
- Which root tasks remain host bootstrap and which move into images?
- Which systemd root assumptions must become `systemd --user` proof gates?
- Which backup/restore and monitoring commands become tools/check images?

**Exit criteria:** bedrock inventory complete; target responsibilities assigned;
T0-T3 gates defined; rootless reboot and restore proof are explicit.

## Scenario B — Already containerized but not operable

**Signals:** `Containerfile` or `Dockerfile` exists, but no public CLI, no
Quadlet, weak docs, no backup/restore proof, healthcheck only checks process.

**Primary agents:** Field Operator Ethnographer, Alloy Architect, Seismic
Observability Analyst, Assay Gate Engineer, Fracture Risk Analyst.

**Handoffs:** `cli-audit-sync`, `cli-forge-pipeline`, `cli-audit-test`,
`cli-forge-doc`.

**Key questions:**

- What day-2 operations are missing from the product surface?
- Are runtime/init/tools/check responsibilities mixed in one image?
- Are docs claiming capabilities that tests do not prove?

**Exit criteria:** CLI verbs mapped; image responsibilities split or justified;
T3 day-2 and T4 stress/fracture gates added.

## Scenario C — Rootless/Quadlet prototype fails after reboot

**Signals:** manual `podman run` works; `systemctl --user` fails; service works
after login but not after reboot; unknown lingering/user bus behavior.

**Primary agents:** Fault-Line Geomechanic, Assay Gate Engineer, Fracture Risk
Analyst.

**Handoffs:** `cli-forge-infra`, `cli-forge-resilience`.

**Key questions:**

- Which account owns the user service?
- Is lingering required and configured?
- Are Quadlet files installed in the correct user scope?
- Are storage permissions compatible with UID mapping after reboot?

**Exit criteria:** clean-host deploy, rerun, restart, reboot, and log access are
all gates; interactive-session dependency removed or explicitly accepted.

## Scenario D — Monitoring / NRPE migration

**Signals:** `check_*`, `nrpe.cfg`, Nagios/Icinga docs, local shell checks,
monitoring SQL role, custom exit messages.

**Primary agents:** Seismic Observability Analyst, Field Operator Ethnographer,
Fault-Line Geomechanic, Assay Gate Engineer.

**Handoffs:** `cli-audit-shell`, `cli-forge-resilience`, optionally
`cli-forge-lld` for check image/CLI design.

**Key questions:**

- What is local health vs remote monitoring?
- What credentials/roles does monitoring use?
- Are exit codes and messages contractually stable?
- Can remote monitoring fail while local checks remain green?

**Exit criteria:** local and remote monitoring paths documented and tested;
check image or bridge has stable exit semantics; degraded cases covered.

## Scenario E — Backup/restore acceptance

**Signals:** backup script exists, restore is manual or absent, archive naming is
unclear, retention/purge exists, WAL/binlog/journal replay is needed.

**Primary agents:** Reservoir & Recovery Engineer, Assay Gate Engineer, Fracture
Risk Analyst.

**Handoffs:** `cli-forge-resilience`, `cli-audit-shell`, `cli-audit-test`.

**Key questions:**

- Is backup logical, physical, or both?
- Is verification independent from backup creation?
- Is restore proven on a clean host or only the source host?
- What happens with corrupt/missing archives and disk pressure?

**Exit criteria:** backup, verify-backup, restore, and clean-host restore are T3
gates; corrupt/missing archive and retention edge cases are T4 gates.

## Scenario F — Legacy CLI / wrapper consolidation

**Signals:** many scripts, aliases, Make targets, historical operator commands,
compatibility requirements, old scripts still invoked by teams.

**Primary agents:** Field Operator Ethnographer, Tailings Remediation Engineer,
Stratigraphy Archaeologist, Assay Gate Engineer.

**Handoffs:** `cli-audit-shell`, `cli-forge-lld`, `cli-forge-doc`.

**Key questions:**

- Which commands are externally visible contract?
- Which wrappers can call the new CLI unchanged?
- Which wrappers still mutate state directly and must become tailings?
- How will compatibility sunset be proven?

**Exit criteria:** one public CLI; wrappers are adapters only; compatibility test
matrix exists; removal conditions are explicit.

## Scenario G — Air-gapped or regulated deployment

**Signals:** private registry, offline install, signed images, SBOM, strict
change control, compliance acceptance, no internet at runtime.

**Primary agents:** Fault-Line Geomechanic, Alloy Architect, Assay Gate Engineer,
Fracture Risk Analyst.

**Handoffs:** `cli-forge-infra`, `cli-forge-pipeline`, `cli-forge-hld`,
`cli-forge-resilience`, optionally `cli-forge-quorum` for high-stakes agentic
implementation governance.

**Key questions:**

- Are images pinned, mirrored, signed, and available offline?
- Are secret/cert rotation paths documented and testable?
- Are acceptance gates auditable?
- What evidence satisfies production approval?

**Exit criteria:** supply-chain gates exist; offline fresh deploy is tested;
restore does not depend on external network; audit artifacts are produced.

## Scenario H — Multi-agent implementation sprint

**Signals:** user asks to parallelize implementation, create worktrees, run many
agents, use a brigade, or execute the migration backlog.

**Primary agents:** Main skill produces migration backlog; then hand off.

**Handoffs:**

- `cli-forge-chef` for ordinary multi-agent implementation sprints.
- `cli-forge-quorum` for high-stakes/compliance/large-agent-count deterministic
  governance.

**Key questions:**

- Is the migration contract stable enough to implement?
- Which tasks are independent work packages?
- Which files are sensitive zones requiring review?
- Which gates must every work package pass?

**Exit criteria:** backlog decomposed into work packages; gates and owners clear;
implementation orchestration generated by the appropriate skill.

## Scenario I — CI gate design for migration proof

**Signals:** CI exists, but does not build images or run rootless/deploy/recovery
gates; tests are happy-path only.

**Primary agents:** Assay Gate Engineer, Alloy Architect, Fracture Risk Analyst.

**Handoffs:** `cli-forge-pipeline`, `cli-audit-test`, `cli-forge-resilience`.

**Key questions:**

- Which gates run in CI vs local/prod-like host?
- Which gates require privileged or VM runners?
- How are artifacts captured as proof?
- Which stress/fracture tests are safe in CI?

**Exit criteria:** T0/T1 in ordinary CI; T2/T3/T4 in prod-like runner or lab;
manual gates are explicitly named with automation targets.

## Scenario J — Incident-driven hardening

**Signals:** user provides outage, failed migration, broken restore, broken
reboot, monitoring miss, secret rotation failure.

**Primary agents:** relevant domain agent + Fracture Risk Analyst + Assay Gate
Engineer.

**Handoffs:** `cli-forge-resilience`, `cli-audit-drift`, `cli-audit-sync`.

**Key questions:**

- Which bedrock invariant was missing or duplicated?
- Which fault line differed from production?
- Which stress/fracture test would have caught the issue?
- What runbook and anti-regression update closes the loop?

**Exit criteria:** incident converted into M0 memory: blackbox entry, runbook
delta, anti-regression test, clarified invariant.
