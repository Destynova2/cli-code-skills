# The 9 biological models — Details and examples

> **When to read:** During analysis workflow steps 1-9. Each section maps to one audit step. All examples show both GitLab CI and GitHub Actions side by side.

---

## 1. LEAFCUTTER ANTS (Atta / Acromyrmex) — Task Partitioning

**Biology**: Workers do NOT do everything from start to finish. They are specialized:
- **Cutters** (high in the tree) → cut the leaves and *drop* them
- **Gatherers** (on the ground) → pick up what falls into the cache
- **Carriers** → transport back to the nest
- **Processors** → cultivate the fungus

Coordination happens through **stigmergy**: the cache of leaves on the ground IS the signal.
No direct communication. No boss.

**CI Pattern: Stage Specialization + Artifact Cache as Coordinator**

**GitLab CI:**
```yaml
# ❌ Anti-pattern: generalist job (does everything, start to finish)
build-and-test-and-push:
  script:
    - cargo build
    - cargo test
    - docker build
    - docker push

# ✅ Leafcutter pattern: specialization + cache as coordinator
stages: [compile, test, package, deploy]

compile:        # Cutter — produces binary artifacts
  stage: compile
  script: cargo build --release
  artifacts:
    paths: [target/release/myapp]
    expire_in: 1h   # The leaf cache is ephemeral

test-unit:      # Gatherer — consumes artifacts, works in parallel
  stage: test
  needs: [compile]   # Direct dependency, no full-stage wait
  parallel: 4        # 4 identical, replaceable workers
  script: cargo test --shard $CI_NODE_INDEX/$CI_NODE_TOTAL

test-integration:
  stage: test
  needs: [compile]   # Parallel to test-unit, not sequential
  script: cargo test --test integration

package:        # Carrier — assembles the final artifact
  stage: package
  needs: [test-unit, test-integration]
  script: docker build -t myapp:$CI_COMMIT_SHA .
```

**GitHub Actions:**
```yaml
# ❌ Anti-pattern: everything in one job
jobs:
  build-and-test-and-push:
    runs-on: ubuntu-latest
    steps:
      - run: cargo build
      - run: cargo test
      - run: docker build
      - run: docker push

# ✅ Leafcutter pattern: separate jobs + artifacts as coordinator
jobs:
  compile:        # Cutter — produces binary artifacts
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: cargo build --release
      - uses: actions/upload-artifact@v4
        with:
          name: binary
          path: target/release/myapp
          retention-days: 1   # The leaf cache is ephemeral

  test-unit:      # Gatherer — consumes artifacts
    needs: [compile]
    runs-on: ubuntu-latest
    strategy:
      matrix:
        shard: [1, 2, 3, 4]   # 4 identical, replaceable workers
    steps:
      - uses: actions/checkout@v4
      - uses: actions/download-artifact@v4
        with: { name: binary }
      - run: cargo nextest run --partition count:${{ matrix.shard }}/4

  test-integration:
    needs: [compile]   # Parallel to test-unit, not sequential
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: cargo test --test integration

  package:        # Carrier — assembles the final artifact
    needs: [test-unit, test-integration]
    runs-on: ubuntu-latest
    steps:
      - run: docker build -t myapp:${{ github.sha }} .
```

**Derived rules:**
- Each job = one role, one responsibility
- GitLab: `needs:` instead of `stages:` → real DAG, not sequential cascade
- GitHub: `needs:` between jobs → same DAG principle
- Artifacts ARE the communication between jobs (stigmergy)
- Identical, replaceable workers (no local state)

---

## 2. SLIME MOLD (Physarum polycephalum) — Adaptive Path Pruning

**Biology**: The slime mold explores all directions in parallel.
Tubes that find food **grow** (reinforcement).
Unsuccessful tubes **shrink** and vanish.
Result: an optimal network between all food sources, fault-tolerant, with no central brain.

The Tokyo experiment (2010): the slime mold recreated the Tokyo rail network with the same efficiency, the same fault tolerance, at the same cost.

**CI Pattern: Change-Driven Path Selection + Cache Reinforcement**

**GitLab CI:**
```yaml
# The slime mold does not retrace paths it has already explored
# → Only rerun what has changed

.only-if-changed: &only-if-changed
  rules:
    - changes:
        - src/auth/**/*
        - tests/auth/**/*

test-auth:
  <<: *only-if-changed
  script: pytest tests/auth/

# Cache as "slime mold memory" — tubes that were used stay wide
cache:
  key:
    files:
      - Cargo.lock        # Key based on content, not date
  paths:
    - target/
  policy: pull-push       # Reads AND writes → reinforces the path

# Automatic fallback if the primary node fails (fault tolerance)
test-e2e:
  retry:
    max: 2
    when: [runner_system_failure, stuck_or_timeout_failure]
  # Do NOT retry on script_failure (real bug) → the slime mold does not retrace wrong paths
```

**GitHub Actions:**
```yaml
# The slime mold does not retrace paths it has already explored
on:
  push:
    paths:            # Only fires on relevant changes
      - 'src/auth/**'
      - 'tests/auth/**'

jobs:
  test-auth:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: pytest tests/auth/

  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      # Cache as "slime mold memory" — tubes that were used stay wide
      - uses: actions/cache@v4
        with:
          path: |
            ~/.cargo/registry
            target/
          key: rust-${{ hashFiles('Cargo.lock') }}           # Key = content
          restore-keys: rust-                                 # Progressive fallback

  test-e2e:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: nick-fields/retry@v3
        with:
          max_attempts: 2
          retry_on: error              # Retry infra, not logic
          command: cargo test --test e2e
```

**Derived rules:**
- Cache key = hash of content (Cargo.lock, package-lock.json, go.sum), never the branch
- GitLab: `rules: changes:` / GitHub: `on.push.paths` or `dorny/paths-filter` to only test paths that were touched
- Selective retry: infra failure ≠ test failure
- DAG with `needs:` = Physarum topology: each job opens the available paths

---

## 3. ARMY ANTS (Eciton burchellii) — Self-Organizing Parallelism

**Biology**: No queen commands the raids.
Columns self-assemble based on traffic:
- The **flanks** fan out (parallel exploration)
- The **center** carries the spoils (return, high density)
- The ants build **living bridges** from their own bodies to optimize throughput

Total replaceability: a dead ant is stepped over, the flow continues.

**CI Pattern: Fan-out / Fan-in + Ephemeral Runners**

**GitLab CI:**
```yaml
# Fan-out: parallel exploration (raid flanks)
test-browser-chrome:
  stage: test
  needs: [build]
  tags: [ephemeral, linux]   # Disposable runner = replaceable ant

test-browser-firefox:
  stage: test
  needs: [build]
  tags: [ephemeral, linux]

test-browser-safari:
  stage: test
  needs: [build]
  tags: [ephemeral, macos]

# Fan-in: consolidation of spoils (raid center)
report-merge:
  stage: report
  needs:
    - job: test-browser-chrome
      artifacts: true
    - job: test-browser-firefox
      artifacts: true
    - job: test-browser-safari
      artifacts: true
  script: merge-junit-reports coverage/

# Living bridge: intermediate job that keeps the flow going
build-assets:
  stage: prebuild
  needs: []   # Starts immediately, in parallel with compile
  script: npm run build:assets
```

**GitHub Actions:**
```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: cargo build --release
      - uses: actions/upload-artifact@v4
        with: { name: build, path: target/release/ }

  # Fan-out: parallel exploration (raid flanks)
  test-browser:
    needs: [build]
    runs-on: ${{ matrix.os }}
    strategy:
      fail-fast: false          # A dead ant is stepped over, the flow continues
      matrix:
        browser: [chrome, firefox, safari]
        include:
          - browser: chrome
            os: ubuntu-latest   # Disposable runner = replaceable ant
          - browser: firefox
            os: ubuntu-latest
          - browser: safari
            os: macos-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/download-artifact@v4
        with: { name: build }
      - run: npx playwright test --project=${{ matrix.browser }}
      - uses: actions/upload-artifact@v4
        with:
          name: results-${{ matrix.browser }}
          path: test-results/

  # Fan-in: consolidation of spoils (raid center)
  report-merge:
    needs: [test-browser]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/download-artifact@v4
        with: { pattern: results-*, merge-multiple: true }
      - run: merge-junit-reports coverage/

  # Living bridge: job that starts immediately in parallel
  build-assets:
    runs-on: ubuntu-latest    # Starts immediately, in parallel with build
    steps:
      - uses: actions/checkout@v4
      - run: npm run build:assets
      - uses: actions/upload-artifact@v4
        with: { name: assets, path: frontend/dist/ }
```

**Derived rules:**
- GitLab: `tags: [ephemeral]` / GitHub: ephemeral runners (ubuntu-latest, ephemeral self-hosted)
- Maximum fan-out on slow jobs (browser tests, multi-arch tests)
- Fan-in on a single consolidation job
- Any job that CAN start with no dependency must start immediately

---

## 4. HONEYBEES (Apis mellifera) — Dynamic Resource Allocation

**Biology**: Scout bees find a source → waggle dance.
Dance intensity = quality/distance of the source.
Foragers allocate themselves **proportionally** to signal quality.
If a source dries up: no more dance → foragers redistribute.

No central plan. The environmental signal drives allocation.

**CI Pattern: Autoscaling + Priority-based Runner Tagging**

**GitLab CI:**
```yaml
# Allocation signal: tags by compute cost
compile-release:
  tags: [fat-runner, 8cpu, 32gb]  # Forager to the productive flower
  script: cargo build --release

lint:
  tags: [shared, 2cpu]            # Forager to an ordinary flower
  script: cargo clippy

security-scan:
  tags: [fat-runner, 16gb]
  script: trivy image myapp:$CI_COMMIT_SHA

# GitLab Runner autoscaling (runner config, not YAML):
# runners.autoscaler.max_instances = 20
# runners.autoscaler.min_instances = 0  ← scale back to 0 when no dance is happening
# runners.autoscaler.scale_down_idle_time = 5m
```

**GitHub Actions:**
```yaml
jobs:
  compile-release:
    runs-on: ubuntu-latest-16-cores  # Forager to the productive flower
    steps:
      - uses: actions/checkout@v4
      - run: cargo build --release

  lint:
    runs-on: ubuntu-latest           # Forager to an ordinary flower (2 CPU)
    steps:
      - uses: actions/checkout@v4
      - run: cargo clippy

  security-scan:
    runs-on: ubuntu-latest-8-cores
    steps:
      - uses: actions/checkout@v4
      - run: trivy image myapp:${{ github.sha }}

# Concurrency groups: only one dance at a time per branch
# Cancels previous runs (foragers recalled from a dried-up source)
concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true
```

**Derived rules:**
- GitLab: runner tags / GitHub: runner labels (ubuntu-latest-16-cores, self-hosted labels)
- Scale-to-zero between pipelines (zero waste outside activity)
- Critical jobs (release, security) → dedicated non-preemptible runners
- GitLab: queue depth monitoring + HPA / GitHub: `concurrency` groups + `cancel-in-progress`

---

## 5. MYCELIUM — Fault-Tolerant Routing + Nutrient Sharing

**Biology**: Mycelium connects the trees of a forest via an underground network.
It transports nutrients and chemical signals.
If a node dies: the network reroutes around it.
There is no central pipe — redundancy IS the structure.

**CI Pattern: Multi-Registry Fallback + Distributed Caching**

**GitLab CI:**
```yaml
# Primary node dead → reroute to secondary (no SPOF)
pull-image:
  script: |
    docker pull harbor.internal/myapp:base || \
    docker pull registry.fallback.io/myapp:base || \
    docker pull ghcr.io/myapp:base
  # Three mycelium paths, the first live one is used

# Distributed cache: every runner contributes to the network
cache:
  - key: deps-$CI_COMMIT_REF_SLUG
    paths: [.npm/]
    policy: pull-push     # Active node → contributes to the network
  - key: deps-main        # Fallback on main if the branch isn't cached yet
    paths: [.npm/]
    policy: pull          # Secondary node → consumes only

# Artifact sharing across pipelines (inter-tree mycelium)
download-base-artifacts:
  script: |
    curl -L "$BASE_PIPELINE_ARTIFACTS_URL" -o base-artifacts.zip
  # Reuses artifacts from an upstream pipeline (nutrient sharing)
```

**GitHub Actions:**
```yaml
jobs:
  pull-image:
    runs-on: ubuntu-latest
    steps:
      # Three mycelium paths, the first live one is used
      - run: |
          docker pull harbor.internal/myapp:base || \
          docker pull registry.fallback.io/myapp:base || \
          docker pull ghcr.io/myapp:base

  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      # Distributed cache: waterfall branch → main → base
      - uses: actions/cache@v4
        with:
          path: ~/.npm
          key: npm-${{ github.ref_name }}-${{ hashFiles('package-lock.json') }}
          restore-keys: |
            npm-${{ github.ref_name }}-
            npm-main-
            npm-
          # Progressive fallback = mycelium searching for a live path

  # Artifact sharing across workflows (inter-tree mycelium)
  use-upstream-artifacts:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: base-build
          github-token: ${{ secrets.GITHUB_TOKEN }}
          repository: myorg/upstream-repo
          run-id: ${{ inputs.upstream_run_id }}
```

**Derived rules:**
- Never a SPOF on registry, cache, or runner
- Cache waterfall: branch → main → base image
- GitLab: multi-project pipelines / GitHub: `workflow_call` + cross-repo artifact download

---

## 6. MITOSIS (Cell division) — Pipeline Splitting

**Biology**: When a cell grows too big, it divides into two independent daughter cells.
Each daughter has the full DNA but then specializes.
Division is triggered by a **size threshold** (G1/S checkpoint).

**CI Pattern: Workflow Fission**

When a pipeline exceeds ~15 jobs or ~20 minutes, it should split into independent workflows that communicate via artifacts/events, not direct dependencies.

**Mitosis signals** (when to split):
- Pipeline > 15 min critical path
- More than 3 `needs:` chained serially
- Jobs that share no artifacts but are in the same workflow
- Different teams modifying the same jobs

**Split examples:**

```
BEFORE (monolithic):
  build → test → lint → e2e → security → release → deploy → notify

AFTER (mitosis):
  Workflow 1 (CI): build → test → lint → e2e
  Workflow 2 (Release): build-release → push-registry → github-release → homebrew
  Workflow 3 (Security): codeql → semgrep → audit → dependabot

  Communication: workflow_run events, artifacts, GHCR images
```

**GitLab**: `trigger:` for child pipelines, `include:` for shared templates.
**GitHub Actions**: `workflow_run:` events, `workflow_call:` for reuse.

**Mitosis rules:**
- Each daughter workflow must be independently executable
- Shared DNA (config, secrets, cache keys) goes into templates/composites
- Communication between workflows = artifacts or registries (stigmergy), never shared state
- The re-mitosis threshold is recursive: if a daughter grows too big, it splits again

---

## 7. IMMUNE SYSTEM (VDJ Recombination) — Combinatorial Fuzzing

**Biology**: The immune system does not react to threats — it generates ~10¹⁸ different antibody combinations ahead of time by random recombination of gene segments (V, D, J), **before** it has ever encountered the antigen. It explores the space of all possible threats combinatorially, then keeps the B cells capable of recognizing anything useful alive (clonal selection).

The system does not guess which threats will occur. It covers the entire possible space.

**CI Pattern: Fuzzing / Property-Based Testing**

**GitLab CI:**
```yaml
# ❌ Anti-pattern: tests only on hand-written cases
test-unit:
  script: cargo test   # Tests what the dev imagined

# ✅ Immune pattern: combinatorial generation of the input space
fuzz-parser:
  stage: test
  needs: [compile]
  script:
    - cargo fuzz run parser -- -max_total_time=300
    # Generates 10^N random inputs, looks for crashes
    # Like VDJ: explores the space BEFORE encountering the threat
  artifacts:
    paths: [fuzz/artifacts/]   # Crash corpus = immune memory
    when: on_failure
  rules:
    - changes:
        - src/parser/**/*      # Only fuzz what has changed (economy)

property-test:
  stage: test
  needs: [compile]
  script:
    - cargo test --features proptest
    # proptest/quickcheck = structured version of VDJ
    # Generates inputs from properties, not examples
```

**GitHub Actions:**
```yaml
jobs:
  fuzz-parser:
    needs: [compile]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: cargo install cargo-fuzz
      - run: cargo fuzz run parser -- -max_total_time=300
      - uses: actions/upload-artifact@v4
        if: failure()
        with:
          name: fuzz-crashes
          path: fuzz/artifacts/   # Corpus = immune memory

  property-test:
    needs: [compile]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: cargo test --features proptest
```

**Derived rules:**
- Fuzzing explores the input space, not known cases → complements unit tests
- The crash corpus persists across runs (acquired immune memory)
- Property-based testing = structured version: you define invariants, the framework generates cases
- Fuzz on the critical path (parsers, serialization, auth) → where inputs come from outside
- Tools: `cargo-fuzz`, `proptest`, `quickcheck` (Rust) / `Hypothesis` (Python) / `go-fuzz` (Go)

---

## 8. FUNGAL SPORES (Diversity + Dispersal) — Combinatorial Matrix Testing

**Biology**: A fungus under stress produces millions of genetically diverse spores and disperses them in **every direction** — all substrates, all microclimates. No prior selection. Maximum dispersal.
What germinates is what is compatible with that specific context.
The others die — no regret, no overhead.

Difference from army ants: army ants parallelize the **same task** (fan-out). Spores test **different combinations** of environments.

**CI Pattern: Full Combinatorial Matrix**

**GitLab CI:**
```yaml
# ❌ Anti-pattern: test one combination
test:
  image: ubuntu:22.04
  script: cargo test   # Works on my machine™

# ✅ Spore pattern: dispersal over all combinations
test-matrix:
  stage: test
  needs: [compile]
  parallel:
    matrix:
      - OS: [ubuntu-22.04, debian-12, alpine-3.19]
        ARCH: [amd64, arm64]
        RUST_VERSION: ["1.75", "1.76", stable, nightly]
  image: $OS
  script:
    - rustup default $RUST_VERSION
    - cargo test
  # 3 × 2 × 4 = 24 spores launched simultaneously
  allow_failure:
    - RUST_VERSION: nightly   # Spores on hostile terrain → failure allowed
```

**GitHub Actions:**
```yaml
jobs:
  test-matrix:
    needs: [compile]
    runs-on: ${{ matrix.os }}
    strategy:
      fail-fast: false   # No recalling the spores: let everything germinate
      matrix:
        os: [ubuntu-22.04, ubuntu-24.04, macos-latest, windows-latest]
        rust: ["1.75", "1.76", stable, nightly]
        exclude:
          - os: windows-latest
            rust: nightly   # Known sterile combination
        include:
          - os: ubuntu-22.04
            rust: stable
            coverage: true   # Spore marked for special harvesting
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@master
        with: { toolchain: "${{ matrix.rust }}" }
      - run: cargo test
      - if: matrix.coverage
        run: cargo llvm-cov --lcov --output-path lcov.info
    continue-on-error: ${{ matrix.rust == 'nightly' }}
```

**Derived rules:**
- Test ALL supported combinations, not just "on my machine"
- `fail-fast: false` (GitHub) / no blocking `dependencies:` (GitLab) → let all spores germinate
- `allow_failure` / `continue-on-error` for experimental combinations (nightly, beta)
- `exclude:` for known impossible combinations (save on spores)
- The matrix covers: OS × architecture × runtime version × feature flags

---

## 9. TARDIGRADE (Ramazzottius varieornatus) — Chaos Engineering

**Biology**: The tardigrade ("water bear") survives:
- **-272°C** and **+150°C** (cryptobiosis)
- **Radiation** at 1000× the lethal human dose
- **Vacuum of space** (Foton-M3 mission, 2007)
- **Pressure** of 6000 atm (6× the ocean floor)
- **Complete desiccation** for 30 years (anhydrobiosis)

It does not choose its conditions. It is **constitutionally ready** for all of them, including some that do not exist naturally on Earth.

Difference from mycelium: mycelium **routes around** failures (passive resilience). The tardigrade **injects failures** to prove it can survive them (proactive resilience).

**CI Pattern: Chaos Engineering / Fault Injection**

**GitLab CI:**
```yaml
# ❌ Anti-pattern: test only under ideal conditions
test-integration:
  script: cargo test --test integration   # Everything fine when everything is fine

# ✅ Tardigrade pattern: inject extreme conditions
chaos-network:
  stage: chaos
  needs: [test-integration]   # After normal tests
  script:
    # Degraded network (5s latency)
    - toxiproxy-cli toxic add -t latency -a latency=5000 postgres
    - cargo test --test integration
    - toxiproxy-cli toxic reset postgres
    # Cut network (timeout)
    - toxiproxy-cli toxic add -t timeout -a timeout=1 postgres
    - cargo test --test integration_timeout_handling
    - toxiproxy-cli toxic reset postgres
  services:
    - toxiproxy

chaos-resources:
  stage: chaos
  needs: [test-integration]
  script:
    # Disk full
    - fallocate -l 95% /tmp/fill
    - cargo test --test storage_full_handling || true
    - rm /tmp/fill
    # Memory under pressure
    - stress-ng --vm 1 --vm-bytes 90% --timeout 60s &
    - cargo test --test memory_pressure
    - kill %1

chaos-time:
  stage: chaos
  needs: [test-integration]
  script:
    # Clock skew (dead NTP → expired certs, invalid tokens)
    - faketime '2099-01-01' cargo test --test auth_token_handling
    # Time in the past (Y2K-like)
    - faketime '1970-01-01' cargo test --test timestamp_handling
```

**GitHub Actions:**
```yaml
jobs:
  chaos-network:
    needs: [test-integration]
    runs-on: ubuntu-latest
    services:
      toxiproxy:
        image: ghcr.io/shopify/toxiproxy:latest
        ports: [8474:8474]
    steps:
      - uses: actions/checkout@v4
      # 5s network latency
      - run: |
          toxiproxy-cli toxic add -t latency -a latency=5000 postgres
          cargo test --test integration
          toxiproxy-cli toxic reset postgres
      # Network cut
      - run: |
          toxiproxy-cli toxic add -t timeout -a timeout=1 postgres
          cargo test --test integration_timeout_handling

  chaos-resources:
    needs: [test-integration]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      # Disk full
      - run: |
          fallocate -l 10G /tmp/fill
          cargo test --test storage_full_handling || true
          rm /tmp/fill
      # OOM pressure
      - run: |
          stress-ng --vm 1 --vm-bytes 90% --timeout 60s &
          cargo test --test memory_pressure
          kill %1 || true

  chaos-time:
    needs: [test-integration]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: sudo apt-get install -y faketime
      - run: faketime '2099-01-01' cargo test --test auth_token_handling
      - run: faketime '1970-01-01' cargo test --test timestamp_handling
```

**Derived rules:**
- Chaos testing comes AFTER functional tests (don't test chaos on broken code)
- Each injection = one isolated extreme condition (network, disk, memory, time)
- Tests must verify **graceful degradation**, not just "it doesn't crash"
- Tools: Toxiproxy (network), stress-ng (CPU/RAM), fallocate (disk), faketime (clock)
- In production: Chaos Monkey (Netflix), Litmus (K8s), Gremlin (SaaS)
- Never inject chaos without **observability** in place (otherwise you don't know what happened)
