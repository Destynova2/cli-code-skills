# The 8 living-system models — biology + physics for resilience

> **When to read:** During workflow Steps 2–7.
> Use these models to derive tests, not to decorate the report with metaphors.

---

## 1. Genome / DNA — Single source of truth

**Meaning:** operational facts should have one canonical source, then be rendered downstream.

Examples:
- image refs
- ports and bind/listen addresses
- secret names and render paths
- role names and grants
- health endpoints
- storage paths
- release tags

**Failure smell:**
- same fact manually copied across YAML, scripts, docs, wrappers, and CI
- fixes applied in symptoms instead of source of truth
- "works on my machine" because local override silently wins

**Tests to derive:**
- render tests
- schema/contract tests
- grep-based duplication drift checks
- executable documentation checks

---

## 2. Membrane / Boundary conditions — Prod parity

**Meaning:** the system is shaped by its environment. If the membrane is different, the behavior changes.

Check:
- OS, kernel, libc, CPU arch
- runtime/orchestrator
- network topology, DNS, ports, TLS
- filesystem, volumes, labels, permissions
- secrets and identities
- realistic data size and shape
- external dependencies and timeout budgets
- clock, timezone, locale

**Failure smell:**
- dev is toy, prod is real
- scans or unit tests pass, deploy fails
- host path and container path, internal port and exposed port, or local secret and runtime secret are confused

**Tests to derive:**
- parity matrix
- clean-room deploy
- access-path differentiation tests

---

## 3. Homeostasis — Steady-state health

**Meaning:** a production system is a control loop. You need quick signals showing whether it is maintaining safe equilibrium.

Need:
- status
- readiness/health
- first useful logs
- a short set of discriminating probes
- known good paths for operator and external client

**Failure smell:**
- too many logs, no first clue
- "degraded" with no interpretation
- status says healthy but real client path is broken

**Tests to derive:**
- startup smoke
- health smoke
- operator-path smoke
- external-path smoke
- monitoring-path smoke

---

## 4. Immune system — Detect what does not belong

**Meaning:** negative tests, fuzzing, mutation tests, and canaries are the antibodies of the delivery system.

Ask:
- what malformed input should be rejected?
- what unexpected state should fail loudly?
- what drift should be detected before merge?
- which critical assumptions can be intentionally broken?

**Failure smell:**
- only hand-written happy-path tests
- no mutation tests
- no assertions on error quality
- false green pipelines

**Tests to derive:**
- negative tests
- property-based tests
- mutation tests
- config corruption tests
- canary gates

---

## 5. Stress–strain — Budgets and fatigue

**Meaning:** systems fail under load, repetition, or exhausted budgets, not only under syntax errors.

Check:
- CPU, memory, file descriptors, storage
- connection limits
- long-running jobs
- repeated deploys/reruns
- backup duration
- artifact growth

**Failure smell:**
- command works once but not repeatedly
- budget assumptions are undocumented
- heavy tools run in constrained environments without proof

**Tests to derive:**
- soak tests
- repeated deploy/rerun tests
- resource budget checks
- backup size/time budget checks

---

## 6. Phase transition — Threshold effects

**Meaning:** behavior changes qualitatively when a threshold is crossed.

Examples:
- disk almost full → startup or WAL behavior changes
- high latency → retries amplify load
- clock skew → token expiry fails
- max connections → monitoring and health probes degrade
- missing dependency → startup order becomes critical

**Failure smell:**
- tests only cover "normal" values
- no edge-of-envelope validation
- production outages appear only near saturation

**Tests to derive:**
- threshold tests
- degraded-mode tests
- failover tests
- timeout/latency injection

---

## 7. Hysteresis — Dirty state after stress

**Meaning:** after partial failure, rollback, or rerun, the system may not return to the initial state automatically.

Typical residues:
- stale volume or cache
- wrong ownership or labels
- partially rendered secrets
- old generated manifests
- leftover sidecar/process/socket
- stale ACL or migration artifact

**Failure smell:**
- fresh install works, rerun fails
- manual cleanup "fixes it"
- same command behaves differently second time

**Tests to derive:**
- rerun/idempotency tests
- rollback then re-apply
- partial-failure replay
- cleanup verification

---

## 8. Memory cells — Incidents become guardrails

**Meaning:** every incident should produce durable immunity.

Store:
- symptom
- first useful signal
- root cause
- source of truth corrected
- anti-regression test
- prevention rule
- runbook delta

**Failure smell:**
- same confusion reappears in new form
- tribal knowledge in chat only
- no closure criteria

**Artifacts to derive:**
- blackbox entries
- hurdle entries
- runbook updates
- pre/post deploy checklists
- closure criteria
