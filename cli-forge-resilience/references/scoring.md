# Resilience scoring — 15 dimensions

> **When to read:** Workflow Step 8.
> Score each dimension from **0 to 4** using evidence from the repo, docs, scripts, deploy flow, and tests.

## D1 — Contract Genome
- **0:** operational facts are scattered or contradictory
- **2:** some canonical config exists, but drift is still manual or undocumented
- **4:** single source of truth with render/contract tests and explicit downstream consumers

## D2 — Boundary Parity
- **0:** dev/staging are toy environments and key prod conditions are unknown
- **2:** some prod-like assumptions are documented, but major gaps remain implicit
- **4:** parity matrix exists with explicit accepted deviations and compensating tests

## D3 — Build Reproducibility
- **0:** snowflake builds, mutable/pinned inconsistently, no repeatability proof
- **2:** builds are mostly reproducible but lack smoke or version discipline
- **4:** reproducible builds with explicit smoke checks and controlled versioning

## D4 — Fresh Deploy Proof
- **0:** no clean deploy validation
- **2:** deploy tested manually once, but not as a stable guardrail
- **4:** clean deploy on prod-like target is a mandatory proof step

## D5 — Rerun / Hysteresis
- **0:** reruns frequently diverge or require tribal cleanup
- **2:** reruns usually work but edge cases remain undocumented
- **4:** reruns converge cleanly or fail loudly with known cleanup rules

## D6 — Runtime Homeostasis
- **0:** health is vague or purely cosmetic
- **2:** some health checks exist, but they do not discriminate enough
- **4:** clear health probes, startup signals, invariants, and operator/external path checks

## D7 — Network Path Fidelity
- **0:** internal, external, and operator paths are mixed together
- **2:** major paths are known, but not all are validated
- **4:** all critical paths are explicit and each has a dedicated test/probe

## D8 — Secrets / Identity / Roles
- **0:** secrets and roles drift manually; least privilege is unproven
- **2:** secrets are managed but render/use/rotation are not fully verified
- **4:** secrets, identities, and roles are rendered from canonical sources and validated end-to-end

## D9 — Data / Storage / Recovery
- **0:** no recovery proof, storage assumptions undocumented
- **2:** backups exist but restore or recovery drills are weak
- **4:** storage layout, sizing, backup, restore, and rollback are proven by repeatable tests

## D10 — Observability / First Error
- **0:** too much noise, no first useful signal
- **2:** logs and metrics exist but require expert interpretation
- **4:** first useful signal is easy to reach and correlated with likely failure surfaces

## D11 — Operability
- **0:** operations depend on tribal knowledge
- **2:** some wrappers or commands exist, but runbook is incomplete
- **4:** wrappers, runbook, smoke probes, and triage sequence are coherent and executable

## D12 — Immune Tests
- **0:** happy path only
- **2:** some negative tests exist but are not systematic
- **4:** negative, mutation, and property-based tests protect critical contracts

## D13 — Chaos / Degraded Mode
- **0:** no degraded-mode validation
- **2:** limited failure injection exists for a few dependencies
- **4:** latency, pressure, failover, threshold, and partial-failure cases are intentionally exercised

## D14 — CI / Release Gate Convergence
- **0:** CI can be green while runtime is unsafe
- **2:** some scanners or smoke checks exist but important gaps are still soft gates
- **4:** release gates align with real runtime risk; false green is hard to achieve

## D15 — Memory / Runbook / Blackbox
- **0:** incidents disappear after the fix
- **2:** some docs exist but they are partial, stale, or not tied to tests
- **4:** every recurrent issue leaves a runbook delta, blackbox entry, and durable anti-regression guardrail

## Score interpretation

| Score | Verdict |
|---|---|
| **52–60** | Excellent — production-hardened, strong parity, strong memory |
| **45–51** | Good — production-capable, a few gaps remain |
| **36–44** | Acceptable — works, but prod surprises are still likely |
| **24–35** | Weak — important parity and gaps |
| **0–23** | Critical — false confidence risk is high |

## Weighting note

If you need a weighted score, weight these dimensions higher:

- D1 Contract Genome
- D4 Fresh Deploy Proof
- D5 Rerun / Hysteresis
- D8 Secrets / Identity / Roles
- D9 Data / Storage / Recovery
- D14 CI / Release Gate Convergence
- D15 Memory / Runbook / Blackbox
