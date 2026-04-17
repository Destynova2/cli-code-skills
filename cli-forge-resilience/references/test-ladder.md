# Test ladder — T0 to T4 plus M0

> **When to read:** Workflow Step 4.
> The ladder is deliberately close to the incident-blackbox style: low-cost checks first, then higher-fidelity validation.

## T0 — Genome tests
**Goal:** catch drift before runtime.
**Typical checks:**
- lint
- schema validation
- render tests
- static contract assertions
- executable documentation
- grep-based duplication detection
- policy / scanner configuration sanity

**Use when changing:**
- config, templates, manifests, ports, refs, docs, wrappers, CI policies

---

## T1 — Organelle tests
**Goal:** prove the component or image is internally sound.
**Typical checks:**
- image build
- unit/integration tests internal to the service
- in-container smoke
- binary/library presence plus actual execution
- local service smoke

**Use when changing:**
- image, binary, package set, build scripts, internal startup logic

---

## T2 — Tissue tests
**Goal:** prove clean deployment in a prod-like environment.
**Typical checks:**
- fresh deploy on clean VM/host/ephemeral environment
- rendered manifests applied end-to-end
- health after deploy
- initial access proof
- baseline observability proof

**Use when changing:**
- deploy scripts, manifests, orchestration, secrets, registries, storage, ports

---

## T3 — Organism tests
**Goal:** prove life, not just birth.
**Typical checks:**
- rerun / idempotency
- operator wrapper path
- external client path
- monitoring path
- backup and restore smoke
- migrations and rollback smoke
- repeated start/stop
- log retrieval and CSV/structured logs

**Use when changing:**
- operators, wrappers, roles, ACLs, backups, commands, monitoring, storage

---

## T4 — Immune tests
**Goal:** expose the bugs that only production stress reveals.
**Typical checks:**
- negative tests
- fault injection
- network latency/loss
- disk pressure
- time skew
- dependency unavailability
- permission denial
- mutation tests on contracts
- threshold and failover tests

**Use when changing:**
- anything with blast radius, or when prior incidents involved hidden drift, degraded behavior, or false green pipelines

---

## M0 — Memory
**Goal:** make the incident expensive only once.
**Artifacts:**
- runbook delta
- blackbox entry
- hurdle / FAQ entry
- anti-regression checklist update
- closure criteria
- broader matrix rerun note

**Rule:** no incident is closed until memory exists in a durable artifact.
