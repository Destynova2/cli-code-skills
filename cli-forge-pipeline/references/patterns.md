# Les 9 modèles biologiques — Détails et exemples

> **When to read:** During analysis workflow steps 1-9. Each section maps to one audit step. All examples show both GitLab CI and GitHub Actions side by side.

---

## 1. FOURMIS COUPERUSES (Atta / Acromyrmex) — Task Partitioning

**Biologie** : Les ouvrières ne font PAS tout du début à la fin. Elles sont spécialisées :
- **Coupeuses** (en hauteur dans l'arbre) → coupent et *lâchent* les feuilles
- **Ramasseuses** (au sol) → récupèrent ce qui tombe dans le cache
- **Porteuses** → transportent vers le nid
- **Transformatrices** → cultivent le champignon

La coordination se fait par **stigmergie** : le cache de feuilles au sol EST le signal.
Pas de communication directe. Pas de chef.

**Pattern CI : Stage Specialization + Artifact Cache as Coordinator**

**GitLab CI :**
```yaml
# ❌ Anti-pattern : généraliste (fait tout, du début à la fin)
build-and-test-and-push:
  script:
    - cargo build
    - cargo test
    - docker build
    - docker push

# ✅ Pattern fourmis : spécialisation + cache comme coordinateur
stages: [compile, test, package, deploy]

compile:        # Coupeuse — produit des artefacts binaires
  stage: compile
  script: cargo build --release
  artifacts:
    paths: [target/release/myapp]
    expire_in: 1h   # Le cache de feuilles est éphémère

test-unit:      # Ramasseuse — consomme les artefacts, travaille en parallèle
  stage: test
  needs: [compile]   # Dépendance directe, pas d'attente de stage complet
  parallel: 4        # 4 ouvrières identiques, remplaçables
  script: cargo test --shard $CI_NODE_INDEX/$CI_NODE_TOTAL

test-integration:
  stage: test
  needs: [compile]   # Parallèle à test-unit, pas séquentiel
  script: cargo test --test integration

package:        # Porteuse — assemble l'artifact final
  stage: package
  needs: [test-unit, test-integration]
  script: docker build -t myapp:$CI_COMMIT_SHA .
```

**GitHub Actions :**
```yaml
# ❌ Anti-pattern : tout dans un seul job
jobs:
  build-and-test-and-push:
    runs-on: ubuntu-latest
    steps:
      - run: cargo build
      - run: cargo test
      - run: docker build
      - run: docker push

# ✅ Pattern fourmis : jobs séparés + artifacts comme coordinateur
jobs:
  compile:        # Coupeuse — produit des artefacts binaires
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: cargo build --release
      - uses: actions/upload-artifact@v4
        with:
          name: binary
          path: target/release/myapp
          retention-days: 1   # Le cache de feuilles est éphémère

  test-unit:      # Ramasseuse — consomme les artefacts
    needs: [compile]
    runs-on: ubuntu-latest
    strategy:
      matrix:
        shard: [1, 2, 3, 4]   # 4 ouvrières identiques, remplaçables
    steps:
      - uses: actions/checkout@v4
      - uses: actions/download-artifact@v4
        with: { name: binary }
      - run: cargo nextest run --partition count:${{ matrix.shard }}/4

  test-integration:
    needs: [compile]   # Parallèle à test-unit, pas séquentiel
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: cargo test --test integration

  package:        # Porteuse — assemble l'artifact final
    needs: [test-unit, test-integration]
    runs-on: ubuntu-latest
    steps:
      - run: docker build -t myapp:${{ github.sha }} .
```

**Règles dérivées :**
- Chaque job = un rôle, une responsabilité
- GitLab: `needs:` au lieu de `stages:` → DAG réel, pas cascade séquentielle
- GitHub: `needs:` entre jobs → même principe de DAG
- Les artifacts SONT la communication entre jobs (stigmergie)
- Workers identiques et remplaçables (pas de state local)

---

## 2. MYXOMYCÈTE (Physarum polycephalum) — Adaptive Path Pruning

**Biologie** : Le blob explore toutes les directions en parallèle.
Les tubes qui trouvent de la nourriture **grossissent** (reinforcement).
Les tubes infructueux **rétrécissent** et disparaissent.
Résultat : réseau optimal entre toutes les sources de nourriture, fault-tolerant, sans cerveau central.

L'expérience Tokyo (2010) : le blob a recréé le réseau ferroviaire de Tokyo avec la même efficacité, la même tolérance aux pannes, au même coût.

**Pattern CI : Change-Driven Path Selection + Cache Reinforcement**

**GitLab CI :**
```yaml
# Le blob ne retrace pas les chemins déjà explorés
# → Ne relancer que ce qui a changé

.only-if-changed: &only-if-changed
  rules:
    - changes:
        - src/auth/**/*
        - tests/auth/**/*

test-auth:
  <<: *only-if-changed
  script: pytest tests/auth/

# Cache comme "mémoire du blob" — les tubes qui ont servi restent larges
cache:
  key:
    files:
      - Cargo.lock        # Clé basée sur le contenu, pas la date
  paths:
    - target/
  policy: pull-push       # Lit ET écrit → renforce le chemin

# Fallback automatique si le noeud primaire tombe (fault tolerance)
test-e2e:
  retry:
    max: 2
    when: [runner_system_failure, stuck_or_timeout_failure]
  # Ne retry PAS sur script_failure (vrai bug) → le blob ne retrace pas les mauvais chemins
```

**GitHub Actions :**
```yaml
# Le blob ne retrace pas les chemins déjà explorés
on:
  push:
    paths:            # Ne déclenche que si changement pertinent
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
      # Cache comme "mémoire du blob" — les tubes qui ont servi restent larges
      - uses: actions/cache@v4
        with:
          path: |
            ~/.cargo/registry
            target/
          key: rust-${{ hashFiles('Cargo.lock') }}           # Clé = contenu
          restore-keys: rust-                                 # Fallback progressif

  test-e2e:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: nick-fields/retry@v3
        with:
          max_attempts: 2
          retry_on: error              # Retry infra, pas logique
          command: cargo test --test e2e
```

**Règles dérivées :**
- Cache key = hash du contenu (Cargo.lock, package-lock.json, go.sum), jamais la branche
- GitLab: `rules: changes:` / GitHub: `on.push.paths` ou `dorny/paths-filter` pour ne tester que les chemins touchés
- Retry sélectif : infra failure ≠ test failure
- DAG avec `needs:` = topologie Physarum : chaque job ouvre les chemins disponibles

---

## 3. FOURMIS LÉGIONNAIRES (Eciton burchellii) — Self-Organizing Parallelism

**Biologie** : Pas de queen qui commande les raids.
Les colonnes se forment spontanément selon le trafic :
- Les **flancs** avancent en éventail (exploration parallèle)
- Le **centre** transporte le butin (retour, haute densité)
- Les fourmis construisent des **ponts vivants** avec leurs propres corps pour optimiser le débit

Remplaçabilité totale : une fourmi morte est enjambée, le flux continue.

**Pattern CI : Fan-out / Fan-in + Ephemeral Runners**

**GitLab CI :**
```yaml
# Fan-out : exploration parallèle (flancs du raid)
test-browser-chrome:
  stage: test
  needs: [build]
  tags: [ephemeral, linux]   # Runner jetable = fourmi remplaçable

test-browser-firefox:
  stage: test
  needs: [build]
  tags: [ephemeral, linux]

test-browser-safari:
  stage: test
  needs: [build]
  tags: [ephemeral, macos]

# Fan-in : consolidation du butin (centre du raid)
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

# Pont vivant : job intermédiaire qui maintient le flux
build-assets:
  stage: prebuild
  needs: []   # Démarre immédiatement, en parallèle du compile
  script: npm run build:assets
```

**GitHub Actions :**
```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: cargo build --release
      - uses: actions/upload-artifact@v4
        with: { name: build, path: target/release/ }

  # Fan-out : exploration parallèle (flancs du raid)
  test-browser:
    needs: [build]
    runs-on: ${{ matrix.os }}
    strategy:
      fail-fast: false          # Une fourmi morte est enjambée, le flux continue
      matrix:
        browser: [chrome, firefox, safari]
        include:
          - browser: chrome
            os: ubuntu-latest   # Runner jetable = fourmi remplaçable
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

  # Fan-in : consolidation du butin (centre du raid)
  report-merge:
    needs: [test-browser]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/download-artifact@v4
        with: { pattern: results-*, merge-multiple: true }
      - run: merge-junit-reports coverage/

  # Pont vivant : job qui démarre immédiatement en parallèle
  build-assets:
    runs-on: ubuntu-latest    # Démarre immédiatement, en parallèle du build
    steps:
      - uses: actions/checkout@v4
      - run: npm run build:assets
      - uses: actions/upload-artifact@v4
        with: { name: assets, path: frontend/dist/ }
```

**Règles dérivées :**
- GitLab: `tags: [ephemeral]` / GitHub: runners éphémères (ubuntu-latest, self-hosted éphéméraux)
- Fan-out maximal sur les jobs lents (tests navigateurs, tests multi-arch)
- Fan-in sur un seul job de consolidation
- Tout job qui peut démarrer SANS dépendance doit démarrer immédiatement

---

## 4. ABEILLES (Apis mellifera) — Dynamic Resource Allocation

**Biologie** : Les abeilles éclaireuses trouvent une source → danse frétillante.
L'intensité de la danse = qualité/distance de la source.
Les butineuses s'allouent **proportionnellement** à la qualité du signal.
Si une source tarit : plus de danse → les butineuses se redistribuent.

Pas de plan central. Le signal dans l'environnement pilote l'allocation.

**Pattern CI : Autoscaling + Priority-based Runner Tagging**

**GitLab CI :**
```yaml
# Signal d'allocation : tags par coût de calcul
compile-release:
  tags: [fat-runner, 8cpu, 32gb]  # Butineuse vers la fleur productive
  script: cargo build --release

lint:
  tags: [shared, 2cpu]            # Butineuse vers fleur ordinaire
  script: cargo clippy

security-scan:
  tags: [fat-runner, 16gb]
  script: trivy image myapp:$CI_COMMIT_SHA

# Autoscaling GitLab Runner (configuration runner, pas YAML) :
# runners.autoscaler.max_instances = 20
# runners.autoscaler.min_instances = 0  ← retour à 0 quand pas de danse
# runners.autoscaler.scale_down_idle_time = 5m
```

**GitHub Actions :**
```yaml
jobs:
  compile-release:
    runs-on: ubuntu-latest-16-cores  # Butineuse vers la fleur productive
    steps:
      - uses: actions/checkout@v4
      - run: cargo build --release

  lint:
    runs-on: ubuntu-latest           # Butineuse vers fleur ordinaire (2 CPU)
    steps:
      - uses: actions/checkout@v4
      - run: cargo clippy

  security-scan:
    runs-on: ubuntu-latest-8-cores
    steps:
      - uses: actions/checkout@v4
      - run: trivy image myapp:${{ github.sha }}

# Concurrency groups : une seule danse à la fois par branche
# Annule les runs précédents (butineuses rappelées de la source tarie)
concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true
```

**Règles dérivées :**
- GitLab: tags runners / GitHub: runner labels (ubuntu-latest-16-cores, self-hosted labels)
- Scale-to-zero entre les pipelines (zéro gaspillage hors activité)
- Jobs critiques (release, sécurité) → runners dédiés non-préemptibles
- GitLab: monitoring queue depth + HPA / GitHub: `concurrency` groups + `cancel-in-progress`

---

## 5. MYCELIUM — Fault-Tolerant Routing + Nutrient Sharing

**Biologie** : Le mycélium relie les arbres d'une forêt via un réseau souterrain.
Il transporte nutriments et signaux chimiques.
Si un noeud meurt : le réseau reroute autour.
Il n'y a pas de tuyau central — la redondance EST la structure.

**Pattern CI : Multi-Registry Fallback + Distributed Caching**

**GitLab CI :**
```yaml
# Noeud primaire mort → reroute vers secondaire (pas de SPOF)
pull-image:
  script: |
    docker pull harbor.internal/myapp:base || \
    docker pull registry.fallback.io/myapp:base || \
    docker pull ghcr.io/myapp:base
  # Trois chemins mycélium, le premier vivant est utilisé

# Cache distribué : chaque runner contribue au réseau
cache:
  - key: deps-$CI_COMMIT_REF_SLUG
    paths: [.npm/]
    policy: pull-push     # Noeud actif → contribue au réseau
  - key: deps-main        # Fallback sur main si branche pas encore cachée
    paths: [.npm/]
    policy: pull          # Noeud secondaire → consomme seulement

# Artifact sharing entre pipelines (mycélium inter-arbres)
download-base-artifacts:
  script: |
    curl -L "$BASE_PIPELINE_ARTIFACTS_URL" -o base-artifacts.zip
  # Réutilise les artifacts d'un pipeline upstream (partage de nutriments)
```

**GitHub Actions :**
```yaml
jobs:
  pull-image:
    runs-on: ubuntu-latest
    steps:
      # Trois chemins mycélium, le premier vivant est utilisé
      - run: |
          docker pull harbor.internal/myapp:base || \
          docker pull registry.fallback.io/myapp:base || \
          docker pull ghcr.io/myapp:base

  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      # Cache distribué : waterfall branche → main → base
      - uses: actions/cache@v4
        with:
          path: ~/.npm
          key: npm-${{ github.ref_name }}-${{ hashFiles('package-lock.json') }}
          restore-keys: |
            npm-${{ github.ref_name }}-
            npm-main-
            npm-
          # Fallback progressif = mycélium qui cherche le chemin vivant

  # Artifact sharing entre workflows (mycélium inter-arbres)
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

**Règles dérivées :**
- Jamais de SPOF sur registry, cache, ou runner
- Cache waterfall : branche → main → image de base
- GitLab: multi-project pipelines / GitHub: `workflow_call` + cross-repo artifact download

---

## 6. MITOSE (Division cellulaire) — Pipeline Splitting

**Biologie** : Quand une cellule devient trop grosse, elle se divise en deux cellules
filles indépendantes. Chaque fille a l'ADN complet mais se spécialise ensuite.
La division est déclenchée par un **seuil de taille** (checkpoint G1/S).

**Pattern CI : Workflow Fission**

Quand un pipeline dépasse ~15 jobs ou ~20 minutes, il doit se diviser en workflows
indépendants qui communiquent par artifacts/events, pas par dépendances directes.

**Signaux de mitose** (quand diviser) :
- Pipeline > 15 min de chemin critique
- Plus de 3 `needs:` chaînés en série
- Des jobs qui ne partagent aucun artifact mais sont dans le même workflow
- Équipes différentes qui modifient les mêmes jobs

**Exemples de division :**

```
AVANT (monolithique) :
  build → test → lint → e2e → security → release → deploy → notify

APRÈS (mitose) :
  Workflow 1 (CI) : build → test → lint → e2e
  Workflow 2 (Release) : build-release → push-registry → github-release → homebrew
  Workflow 3 (Security) : codeql → semgrep → audit → dependabot

  Communication : workflow_run events, artifacts, GHCR images
```

**GitLab** : `trigger:` pour pipelines enfants, `include:` pour templates partagés.
**GitHub Actions** : `workflow_run:` events, `workflow_call:` pour réutilisation.

**Règles de mitose :**
- Chaque workflow fille doit être exécutable indépendamment
- L'ADN partagé (config, secrets, cache keys) va dans des templates/composites
- La communication entre workflows = artifacts ou registries (stigmergie), jamais state partagé
- Le seuil de re-mitose est récursif : si une fille grossit trop, elle se divise aussi

---

## 7. SYSTÈME IMMUNITAIRE (Recombinaison VDJ) — Combinatorial Fuzzing

**Biologie** : Le système immunitaire ne réagit pas aux menaces — il génère à l'avance
~10¹⁸ combinaisons d'anticorps différents par recombinaison aléatoire de segments géniques
(V, D, J), **avant** d'avoir jamais rencontré l'antigène. Il explore l'espace de toutes les
menaces possibles de manière combinatoire, puis maintient en vie les cellules B capables de
reconnaître quelque chose d'utile (sélection clonale).

Le système ne devine pas quelles menaces vont survenir. Il couvre tout l'espace possible.

**Pattern CI : Fuzzing / Property-Based Testing**

**GitLab CI :**
```yaml
# ❌ Anti-pattern : tests uniquement sur des cas écrits à la main
test-unit:
  script: cargo test   # Teste ce que le dev a imaginé

# ✅ Pattern immunitaire : génération combinatoire de l'espace d'inputs
fuzz-parser:
  stage: test
  needs: [compile]
  script:
    - cargo fuzz run parser -- -max_total_time=300
    # Génère 10^N inputs aléatoires, cherche les crashs
    # Comme le VDJ : explore l'espace AVANT de rencontrer la menace
  artifacts:
    paths: [fuzz/artifacts/]   # Corpus de crashs = mémoire immunitaire
    when: on_failure
  rules:
    - changes:
        - src/parser/**/*      # Ne fuzz que ce qui a changé (économie)

property-test:
  stage: test
  needs: [compile]
  script:
    - cargo test --features proptest
    # proptest/quickcheck = version structurée du VDJ
    # Génère des inputs selon des propriétés, pas des exemples
```

**GitHub Actions :**
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
          path: fuzz/artifacts/   # Corpus = mémoire immunitaire

  property-test:
    needs: [compile]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: cargo test --features proptest
```

**Règles dérivées :**
- Le fuzzing explore l'espace des inputs, pas les cas connus → complète les tests unitaires
- Le corpus de crashs persiste entre runs (mémoire immunitaire acquise)
- Property-based testing = version structurée : tu définis les invariants, le framework génère les cas
- Fuzzer sur le chemin critique (parsers, sérialisation, auth) → là où les inputs viennent de l'extérieur
- Outils : `cargo-fuzz`, `proptest`, `quickcheck` (Rust) / `Hypothesis` (Python) / `go-fuzz` (Go)

---

## 8. SPORES FONGIQUES (Diversité + Dispersion) — Combinatorial Matrix Testing

**Biologie** : Un champignon sous stress produit des millions de spores génétiquement
diversifiées et les disperse dans **toutes les directions** — tous les substrats, tous les
microclimats. Aucune sélection préalable. Dispersion maximale.
Ce qui germera, c'est ce qui est compatible avec ce contexte précis.
Les autres meurent — sans regret, sans overhead.

Différence avec les fourmis légionnaires : les légionnaires parallélisent la **même tâche**
(fan-out). Les spores testent des **combinaisons différentes** d'environnements.

**Pattern CI : Full Combinatorial Matrix**

**GitLab CI :**
```yaml
# ❌ Anti-pattern : tester une seule combinaison
test:
  image: ubuntu:22.04
  script: cargo test   # Marche chez moi™

# ✅ Pattern spores : dispersion sur toutes les combinaisons
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
  # 3 × 2 × 4 = 24 spores lancées simultanément
  allow_failure:
    - RUST_VERSION: nightly   # Spores sur terrain hostile → échec accepté
```

**GitHub Actions :**
```yaml
jobs:
  test-matrix:
    needs: [compile]
    runs-on: ${{ matrix.os }}
    strategy:
      fail-fast: false   # Pas de rappel des spores : laisser tout germer
      matrix:
        os: [ubuntu-22.04, ubuntu-24.04, macos-latest, windows-latest]
        rust: ["1.75", "1.76", stable, nightly]
        exclude:
          - os: windows-latest
            rust: nightly   # Combinaison connue pour être stérile
        include:
          - os: ubuntu-22.04
            rust: stable
            coverage: true   # Spore marquée pour récolte spéciale
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@master
        with: { toolchain: "${{ matrix.rust }}" }
      - run: cargo test
      - if: matrix.coverage
        run: cargo llvm-cov --lcov --output-path lcov.info
    continue-on-error: ${{ matrix.rust == 'nightly' }}
```

**Règles dérivées :**
- Tester TOUTES les combinaisons supportées, pas juste "chez moi"
- `fail-fast: false` (GitHub) / pas de `dependencies:` bloquant (GitLab) → laisser toutes les spores germer
- `allow_failure` / `continue-on-error` pour les combinaisons expérimentales (nightly, beta)
- `exclude:` pour les combinaisons connues impossibles (économie de spores)
- La matrice couvre : OS × architecture × version runtime × feature flags

---

## 9. TARDIGRADE (Ramazzottius varieornatus) — Chaos Engineering

**Biologie** : Le tardigrade ("ourson d'eau") survit à :
- **-272°C** et **+150°C** (cryptobiose)
- **Radiation** 1000× la dose létale humaine
- **Vide spatial** (mission Foton-M3, 2007)
- **Pression** 6000 atm (6× le fond océanique)
- **Dessiccation** complète pendant 30 ans (anhydrobiose)

Il ne choisit pas ses conditions. Il est **constitutionnellement prêt** pour toutes,
y compris celles qui n'existent pas naturellement sur Terre.

Différence avec le mycélium : le mycélium **route autour** des pannes (résilience passive).
Le tardigrade **injecte les pannes** pour prouver qu'on y survit (résilience proactive).

**Pattern CI : Chaos Engineering / Fault Injection**

**GitLab CI :**
```yaml
# ❌ Anti-pattern : tester uniquement en conditions idéales
test-integration:
  script: cargo test --test integration   # Tout va bien quand tout va bien

# ✅ Pattern tardigrade : injecter les conditions extrêmes
chaos-network:
  stage: chaos
  needs: [test-integration]   # Après les tests normaux
  script:
    # Réseau dégradé (latence 5s)
    - toxiproxy-cli toxic add -t latency -a latency=5000 postgres
    - cargo test --test integration
    - toxiproxy-cli toxic reset postgres
    # Réseau coupé (timeout)
    - toxiproxy-cli toxic add -t timeout -a timeout=1 postgres
    - cargo test --test integration_timeout_handling
    - toxiproxy-cli toxic reset postgres
  services:
    - toxiproxy

chaos-resources:
  stage: chaos
  needs: [test-integration]
  script:
    # Disque plein
    - fallocate -l 95% /tmp/fill
    - cargo test --test storage_full_handling || true
    - rm /tmp/fill
    # Mémoire sous pression
    - stress-ng --vm 1 --vm-bytes 90% --timeout 60s &
    - cargo test --test memory_pressure
    - kill %1

chaos-time:
  stage: chaos
  needs: [test-integration]
  script:
    # Clock skew (NTP mort → certificats expirés, tokens invalides)
    - faketime '2099-01-01' cargo test --test auth_token_handling
    # Heure dans le passé (Y2K-like)
    - faketime '1970-01-01' cargo test --test timestamp_handling
```

**GitHub Actions :**
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
      # Latence réseau 5s
      - run: |
          toxiproxy-cli toxic add -t latency -a latency=5000 postgres
          cargo test --test integration
          toxiproxy-cli toxic reset postgres
      # Coupure réseau
      - run: |
          toxiproxy-cli toxic add -t timeout -a timeout=1 postgres
          cargo test --test integration_timeout_handling

  chaos-resources:
    needs: [test-integration]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      # Disque plein
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

**Règles dérivées :**
- Le chaos testing vient APRÈS les tests fonctionnels (on ne teste pas le chaos sur du code cassé)
- Chaque injection = une condition extrême isolée (réseau, disque, mémoire, temps)
- Les tests doivent vérifier la **dégradation gracieuse**, pas juste "ça ne crash pas"
- Outils : Toxiproxy (réseau), stress-ng (CPU/RAM), fallocate (disque), faketime (horloge)
- En production : Chaos Monkey (Netflix), Litmus (K8s), Gremlin (SaaS)
- Ne jamais injecter du chaos sans **observabilité** en place (sinon on ne sait pas ce qui s'est passé)
