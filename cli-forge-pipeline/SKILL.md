---
name: cli-forge-pipeline
description: >
  Expert CI/CD pipeline optimizer using biomimetic patterns from nature: leafcutter ants
  (task partitioning), slime mold (adaptive path optimization), army ants (self-organizing
  parallelism), honeybees (dynamic resource allocation), and mycelium (fault-tolerant routing).
  Works with any CI system — examples cover both GitLab CI and GitHub Actions.
  Use this skill whenever the user asks to optimize, design, review, speed up, parallelize,
  or fix a CI/CD pipeline. Also triggers on: "pipeline trop lent", "flaky tests", "runners",
  "artifacts", "cache CI", "build parallèle", "GitLab CI", "GitHub Actions", "pipeline design",
  "réduire le temps de build", DAG pipelines, job dependencies, or any request mixing
  infrastructure + automation + deployment. Use it even when the user just pastes a YAML
  pipeline without asking explicitly.
---

# CI Pipeline Optimizer — Biomimétique

> *"La colonie de fourmis couperuses de feuilles n'a pas de chef de projet.
> Pourtant elle déplace 15% de la végétation tropicale de manière optimale."*

> **Note** : Les patterns biologiques ci-dessous sont universels. Chaque section montre
> l'implémentation **GitLab CI** et **GitHub Actions** côte à côte. Les principes s'appliquent
> à tout système CI (Jenkins, CircleCI, Buildkite, Dagger, etc.) — seule la syntaxe change.

## Les 9 modèles biologiques → patterns CI

### 1. FOURMIS COUPERUSES (Atta / Acromyrmex) — Task Partitioning

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

**Regles derivees :**
- Chaque job = un role, une responsabilite
- GitLab: `needs:` au lieu de `stages:` → DAG reel, pas cascade sequentielle
- GitHub: `needs:` entre jobs → meme principe de DAG
- Les artifacts SONT la communication entre jobs (stigmergie)
- Workers identiques et remplacables (pas de state local)

---

### 2. MYXOMYCETE (Physarum polycephalum) — Adaptive Path Pruning

**Biologie** : Le blob explore toutes les directions en parallele.
Les tubes qui trouvent de la nourriture **grossissent** (reinforcement).
Les tubes infructueux **retrecissent** et disparaissent.
Resultat : reseau optimal entre toutes les sources de nourriture, fault-tolerant, sans cerveau central.

L'experience Tokyo (2010) : le blob a recree le reseau ferroviaire de Tokyo avec la meme efficacite, la meme tolerance aux pannes, au meme cout.

**Pattern CI : Change-Driven Path Selection + Cache Reinforcement**

**GitLab CI :**
```yaml
# Le blob ne retrace pas les chemins deja explores
# → Ne relancer que ce qui a change

.only-if-changed: &only-if-changed
  rules:
    - changes:
        - src/auth/**/*
        - tests/auth/**/*

test-auth:
  <<: *only-if-changed
  script: pytest tests/auth/

# Cache comme "memoire du blob" — les tubes qui ont servi restent larges
cache:
  key:
    files:
      - Cargo.lock        # Cle basee sur le contenu, pas la date
  paths:
    - target/
  policy: pull-push       # Lit ET ecrit → renforce le chemin

# Fallback automatique si le noeud primaire tombe (fault tolerance)
test-e2e:
  retry:
    max: 2
    when: [runner_system_failure, stuck_or_timeout_failure]
  # Ne retry PAS sur script_failure (vrai bug) → le blob ne retrace pas les mauvais chemins
```

**GitHub Actions :**
```yaml
# Le blob ne retrace pas les chemins deja explores
on:
  push:
    paths:            # Ne declenche que si changement pertinent
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
      # Cache comme "memoire du blob" — les tubes qui ont servi restent larges
      - uses: actions/cache@v4
        with:
          path: |
            ~/.cargo/registry
            target/
          key: rust-${{ hashFiles('Cargo.lock') }}           # Cle = contenu
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

**Regles derivees :**
- Cache key = hash du contenu (Cargo.lock, package-lock.json, go.sum), jamais la branche
- GitLab: `rules: changes:` / GitHub: `on.push.paths` ou `dorny/paths-filter` pour ne tester que les chemins touches
- Retry selectif : infra failure ≠ test failure
- DAG avec `needs:` = topologie Physarum : chaque job ouvre les chemins disponibles

---

### 3. FOURMIS LEGIONNAIRES (Eciton burchellii) — Self-Organizing Parallelism

**Biologie** : Pas de queen qui commande les raids.
Les colonnes se forment spontanement selon le trafic :
- Les **flancs** avancent en eventail (exploration parallele)
- Le **centre** transporte le butin (retour, haute densite)
- Les fourmis construisent des **ponts vivants** avec leurs propres corps pour optimiser le debit

Remplacabilite totale : une fourmi morte est enjambee, le flux continue.

**Pattern CI : Fan-out / Fan-in + Ephemeral Runners**

**GitLab CI :**
```yaml
# Fan-out : exploration parallele (flancs du raid)
test-browser-chrome:
  stage: test
  needs: [build]
  tags: [ephemeral, linux]   # Runner jetable = fourmi remplacable

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

# Pont vivant : job intermediaire qui maintient le flux
build-assets:
  stage: prebuild
  needs: []   # Demarre immediatement, en parallele du compile
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

  # Fan-out : exploration parallele (flancs du raid)
  test-browser:
    needs: [build]
    runs-on: ${{ matrix.os }}
    strategy:
      fail-fast: false          # Une fourmi morte est enjambee, le flux continue
      matrix:
        browser: [chrome, firefox, safari]
        include:
          - browser: chrome
            os: ubuntu-latest   # Runner jetable = fourmi remplacable
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

  # Pont vivant : job qui demarre immediatement en parallele
  build-assets:
    runs-on: ubuntu-latest    # Demarre immediatement, en parallele du build
    steps:
      - uses: actions/checkout@v4
      - run: npm run build:assets
      - uses: actions/upload-artifact@v4
        with: { name: assets, path: frontend/dist/ }
```

**Regles derivees :**
- GitLab: `tags: [ephemeral]` / GitHub: runners ephemeres (ubuntu-latest, self-hosted ephemeraux)
- Fan-out maximal sur les jobs lents (tests navigateurs, tests multi-arch)
- Fan-in sur un seul job de consolidation
- Tout job qui peut demarrer SANS dependance doit demarrer immediatement

---

### 4. ABEILLES (Apis mellifera) — Dynamic Resource Allocation

**Biologie** : Les abeilles eclaireuses trouvent une source → danse fretillante.
L'intensite de la danse = qualite/distance de la source.
Les butineuses s'allouent **proportionnellement** a la qualite du signal.
Si une source tarit : plus de danse → les butineuses se redistribuent.

Pas de plan central. Le signal dans l'environnement pilote l'allocation.

**Pattern CI : Autoscaling + Priority-based Runner Tagging**

**GitLab CI :**
```yaml
# Signal d'allocation : tags par cout de calcul
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
# runners.autoscaler.min_instances = 0  ← retour a 0 quand pas de danse
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

# Concurrency groups : une seule danse a la fois par branche
# Annule les runs precedents (butineuses rappelees de la source tarie)
concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true
```

**Regles derivees :**
- GitLab: tags runners / GitHub: runner labels (ubuntu-latest-16-cores, self-hosted labels)
- Scale-to-zero entre les pipelines (zero gaspillage hors activite)
- Jobs critiques (release, securite) → runners dedies non-preemptibles
- GitLab: monitoring queue depth + HPA / GitHub: `concurrency` groups + `cancel-in-progress`

---

### 5. MYCELIUM — Fault-Tolerant Routing + Nutrient Sharing

**Biologie** : Le mycelium relie les arbres d'une foret via un reseau souterrain.
Il transporte nutriments et signaux chimiques.
Si un noeud meurt : le reseau rerouage autour.
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
  # Trois chemins mycelium, le premier vivant est utilise

# Cache distribue : chaque runner contribue au reseau
cache:
  - key: deps-$CI_COMMIT_REF_SLUG
    paths: [.npm/]
    policy: pull-push     # Noeud actif → contribue au reseau
  - key: deps-main        # Fallback sur main si branche pas encore cachee
    paths: [.npm/]
    policy: pull          # Noeud secondaire → consomme seulement

# Artifact sharing entre pipelines (mycelium inter-arbres)
download-base-artifacts:
  script: |
    curl -L "$BASE_PIPELINE_ARTIFACTS_URL" -o base-artifacts.zip
  # Reutilise les artifacts d'un pipeline upstream (partage de nutriments)
```

**GitHub Actions :**
```yaml
jobs:
  pull-image:
    runs-on: ubuntu-latest
    steps:
      # Trois chemins mycelium, le premier vivant est utilise
      - run: |
          docker pull harbor.internal/myapp:base || \
          docker pull registry.fallback.io/myapp:base || \
          docker pull ghcr.io/myapp:base

  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      # Cache distribue : waterfall branche → main → base
      - uses: actions/cache@v4
        with:
          path: ~/.npm
          key: npm-${{ github.ref_name }}-${{ hashFiles('package-lock.json') }}
          restore-keys: |
            npm-${{ github.ref_name }}-
            npm-main-
            npm-
          # Fallback progressif = mycelium qui cherche le chemin vivant

  # Artifact sharing entre workflows (mycelium inter-arbres)
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

**Regles derivees :**
- Jamais de SPOF sur registry, cache, ou runner
- Cache waterfall : branche → main → image de base
- GitLab: multi-project pipelines / GitHub: `workflow_call` + cross-repo artifact download

---

## Workflow d'analyse d'un pipeline existant

Quand on te donne un pipeline a analyser :

### Etape 1 : Cartographie DAG reel
```
Dessine les dependances reelles entre jobs.
Identifie : jobs sequentiels qui pourraient etre paralleles.
Calcule le chemin critique (Physarum : quel tube est le goulot ?).
```

### Etape 2 : Audit fourmis couperuses
```
Chaque job fait-il UNE chose ?
Les artifacts sont-ils bien definis (communication par stigmergie) ?
Y a-t-il des jobs "generalistes" a decouper ?
```

### Etape 3 : Audit blob
```
Y a-t-il des jobs qui tournent meme sans changement pertinent ?
Les cles de cache sont-elles basees sur le contenu ou le temps ?
Les retries sont-ils selectifs (infra vs. code failure) ?
```

### Etape 4 : Audit legionnaires
```
Quels jobs lents peuvent etre fan-out en parallele ?
Les runners sont-ils ephemeres (zero state local) ?
Y a-t-il des fan-in inutilement precoces ?
```

### Etape 5 : Audit abeilles
```
Les runners sont-ils dimensionnes au bon profil ?
Y a-t-il du scale-to-zero possible ?
Les jobs critiques ont-ils des runners dedies ?
```

### Etape 6 : Audit mycelium
```
Y a-t-il des SPOF (registry, cache, runner unique) ?
Le cache a-t-il un fallback waterfall ?
Les pipelines multi-projets partagent-ils les artifacts couteux ?
```

### Etape 7 : Audit système immunitaire
```
Y a-t-il du fuzzing sur les parsers, la sérialisation, l'auth ?
Les tests utilisent-ils du property-based testing (proptest, Hypothesis) ?
Le corpus de crashs est-il persisté entre runs ?
```

### Etape 8 : Audit spores
```
La matrice couvre-t-elle OS × arch × version × features ?
Les combinaisons nightly/beta sont-elles en allow_failure ?
Y a-t-il des combinaisons impossibles à exclure ?
```

### Etape 9 : Audit tardigrade
```
Y a-t-il des tests en conditions réseau dégradé (latence, coupure) ?
Y a-t-il des tests sous pression ressources (disque, mémoire) ?
Y a-t-il des tests avec clock skew (faketime) ?
La dégradation gracieuse est-elle vérifiée (pas juste "pas de crash") ?
```

---

## Anti-patterns a identifier

| Anti-pattern | Biologie cassee | Fix GitLab | Fix GitHub Actions |
|---|---|---|---|
| Jobs sequentiels sans dependance reelle | Fourmi generaliste | `needs:` DAG direct | `needs:` entre jobs |
| Cache key = branche ou date | Blob sans memoire | `key: files: [Cargo.lock]` | `key: rust-${{ hashFiles('Cargo.lock') }}` |
| Retry sur tout | Blob qui retrace les mauvais chemins | `when: [runner_system_failure]` | `nick-fields/retry` avec `retry_on: error` |
| Rebuild complet sur changement mineur | Blob sans pruning | `rules: changes:` | `on.push.paths` ou `dorny/paths-filter` |
| Runner unique pour tout | Pas de division des castes | `tags:` + autoscaler | Runner labels + larger runners |
| Registry sans fallback | Mycelium sans redondance | `cmd1 \|\| cmd2 \|\| cmd3` | `cmd1 \|\| cmd2 \|\| cmd3` |
| Artifacts enormes transmis partout | Porteuse qui transporte tout a tous | Artifacts scoped, `needs:` selectif | Artifacts nommes + `download-artifact` selectif |
| Tests non-shardes | Fourmis legionnaires sans flancs | `parallel: N` + sharding | `strategy.matrix` + sharding |
| Tests uniquement sur cas écrits à la main | Immunitaire sans VDJ | `cargo fuzz` + `proptest` | `cargo fuzz` + `proptest` |
| Une seule combinaison OS/version | Spore unique sans dispersion | `parallel: matrix:` | `strategy.matrix` combinatoire |
| Pas de test en conditions dégradées | Tardigrade qui n'a jamais quitté la flaque | Toxiproxy + stress-ng + faketime | Toxiproxy + stress-ng + faketime |

---

## Templates de reference

Pour des implementations completes, lire les fichiers dans `references/` :

- `references/gitlab-ci-biomimic.yml` — Pipeline GitLab complet avec tous les patterns
- `references/github-actions-biomimic.yml` — Pipeline GitHub Actions complet avec tous les patterns
- `references/runner-autoscaler.toml` — Config GitLab Runner autoscaler (abeilles)
- `references/cache-strategy.md` — Strategies de cache avancees (blob + mycelium)

---

## Metriques de succes

Un pipeline biomimetique optimal vise :

- **Chemin critique** : < 50% du temps total theorique sequentiel
- **Cache hit rate** : > 80% sur les dependances
- **Runner utilization** : > 70% (pas de surcapacite idle)
- **Flaky test rate** : < 1% (retry selectif, pas masquage)
- **SPOF count** : 0

> Lire `references/cache-strategy.md` pour les patterns de cache avances (Physarum + Mycelium combines).

---

## 🧬 6. MITOSE (Division cellulaire) — Pipeline Splitting

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

## 🧫 7. SYSTÈME IMMUNITAIRE (Recombinaison VDJ) — Combinatorial Fuzzing

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

## 🍄 8. SPORES FONGIQUES (Diversité + Dispersion) — Combinatorial Matrix Testing

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
  # Celles qui tombent sur un sol fertile (compat) germent
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

## 🐻 9. TARDIGRADE (Ramazzottius varieornatus) — Chaos Engineering

**Biologie** : Le tardigrade ("ourson d'eau") survit à :
- **-272°C** et **+150°C** (cryptobiose)
- **Radiation** 1000× la dose létale humaine
- **Vide spatial** (mission Foton-M3, 2007)
- **Pression** 6000 atm (6× le fond océanique)
- **Dessiccation** complète pendant 30 ans (anhydrobiose)

Il ne choisit pas ses conditions. Il est **constitutionnellement prêt** pour toutes,
y compris celles qui n'existent pas naturellement sur Terre.

Différence avec le mycelium : le mycelium **route autour** des pannes (résilience passive).
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

---

## 📊 Pipeline Scoring (15 dimensions)

Scorer un pipeline comme on score un test plan. Chaque dimension vaut 0-4.

| # | Dimension | 0 | 1 | 2 | 3 | 4 |
|---|-----------|---|---|---|---|---|
| D1 | **DAG** | Tout séquentiel | Quelques `needs:` | DAG partiel | DAG complet | DAG + pruning dynamique |
| D2 | **Cache** | Aucun cache | Cache branche | Cache hash lockfile | Cache waterfall | Cache cross-pipeline + GC |
| D3 | **Parallélisme** | 1 runner | 2-3 jobs // | Matrix/sharding | Fan-out adaptatif | Auto-scale + spot instances |
| D4 | **Resilience** | Pas de retry | Retry aveugle | Retry sélectif (infra) | Fallback registry | Multi-provider + self-healing |
| D5 | **Feedback** | > 15 min | 10-15 min | 5-10 min | 2-5 min | < 2 min (smoke) |
| D6 | **Pruning** | Rebuild tout | paths filter | changes + affected | Merge queue | Predictive skip (ML) |
| D7 | **Artifacts** | Tout partagé | Scoped basique | `needs:` sélectif | Size-optimized | Content-addressed (CAS) |
| D8 | **Security** | Aucun scan | 1 scanner | SAST + deps | + secrets + DAST | + SBOM + signing |
| D9 | **Observabilité** | Logs bruts | Durées par job | Métriques custom | Dashboard temps réel | Anomaly detection |
| D10 | **Mitose** | 1 mega-pipeline | 2 workflows | N workflows scopés | Templates partagés | Event-driven mesh |
| D11 | **Coût** | Pas de mesure | Estimation | Budget alerts | Per-team billing | FinOps optimized |
| D12 | **DX** | Config manuelle | Docs | CLI helpers | Self-service portal | GitOps + preview envs |
| D13 | **Fuzzing** | Aucun | Fuzz ponctuel | Fuzz CI sur parsers | + property-based | Corpus persistant + regression |
| D14 | **Matrix** | 1 env | 2-3 combos | OS × version | + arch × features | Full combinatorial + nightly |
| D15 | **Chaos** | Aucun | Retry = seul filet | Toxiproxy en CI | + disk/mem/time | Chaos Monkey prod + observabilité |

**Score cible :** > 45/60 (75%) pour un projet production.

### Scoring rapide (copier-coller)

```
Pipeline: _______________
Date: _______________

D1  DAG:           [ ] 0  [ ] 1  [ ] 2  [ ] 3  [ ] 4
D2  Cache:         [ ] 0  [ ] 1  [ ] 2  [ ] 3  [ ] 4
D3  Parallélisme:  [ ] 0  [ ] 1  [ ] 2  [ ] 3  [ ] 4
D4  Résilience:    [ ] 0  [ ] 1  [ ] 2  [ ] 3  [ ] 4
D5  Feedback:      [ ] 0  [ ] 1  [ ] 2  [ ] 3  [ ] 4
D6  Pruning:       [ ] 0  [ ] 1  [ ] 2  [ ] 3  [ ] 4
D7  Artifacts:     [ ] 0  [ ] 1  [ ] 2  [ ] 3  [ ] 4
D8  Security:      [ ] 0  [ ] 1  [ ] 2  [ ] 3  [ ] 4
D9  Observabilité: [ ] 0  [ ] 1  [ ] 2  [ ] 3  [ ] 4
D10 Mitose:        [ ] 0  [ ] 1  [ ] 2  [ ] 3  [ ] 4
D11 Coût:          [ ] 0  [ ] 1  [ ] 2  [ ] 3  [ ] 4
D12 DX:            [ ] 0  [ ] 1  [ ] 2  [ ] 3  [ ] 4
D13 Fuzzing:       [ ] 0  [ ] 1  [ ] 2  [ ] 3  [ ] 4
D14 Matrix:        [ ] 0  [ ] 1  [ ] 2  [ ] 3  [ ] 4
D15 Chaos:         [ ] 0  [ ] 1  [ ] 2  [ ] 3  [ ] 4

TOTAL: ___/60  (___%)
```

---

## 💀 Pre-Mortem Pipeline

Avant de merger un changement de pipeline, imaginer les échecs :

```
Le pipeline a échoué en production. Que s'est-il passé ?

1. Le cache a été empoisonné (attaquant a push un artefact malveillant)
2. Le runner a été compromis (secret leak via env var)
3. Le registry est down (pas de fallback → deploy bloqué)
4. Un job flaky a été retry-masqué pendant 3 mois (bug en prod)
5. Le pipeline a pris 45 min car le cache a miss (lockfile changé)
6. Cross-workflow dependency a cassé silencieusement (event pas émis)
7. Le DAG a un diamant de dépendances (A→B, A→C, B+C→D, mais C échoue silencieusement)
```

Pour chaque scénario : quelle mitigation existe DÉJÀ dans le pipeline ?
Si aucune → c'est un risque accepté, documentez-le.

---

## 🧪 Pipeline Mutation Testing

Introduire des mutations dans le pipeline YAML et vérifier que les protections détectent :

| Mutation | Attendu | Si ça passe = bug |
|----------|---------|-------------------|
| Supprimer un `needs:` | Job tourne trop tôt, tests échouent | DAG mal configuré |
| Changer la cache key en `always-hit` | Cache empoisonné détecté | Pas de validation cache |
| Supprimer le `paths:` filter | Tout rebuild → lent mais pas cassé | OK (conservative) |
| Mettre `continue-on-error: true` partout | Release passe malgré test failure | Gate manquant |
| Supprimer le job security scan | Release sans scan | Pas de gate security |
| Doubler le `timeout-minutes` | Job lent pas détecté | Pas d'alerte durée |

Si > 2 mutations passent silencieusement, le pipeline a des trous.
