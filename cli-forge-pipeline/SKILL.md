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
argument-hint: "[pipeline-yaml-file-or-ci-directory]"
context: fork
agent: general-purpose
allowed-tools:
  - Read
  - Write
  - Grep
  - Glob
  - Bash
  - Agent
---

> **Optimization:** This skill uses on-demand loading. Heavy content lives in `references/` and is loaded only when needed.

> **Language rule:** Detect the project's primary language (from README, comments, docs, commit messages). Output your report in that language. If the project is bilingual, ask the user which language to use before proceeding.

# CI Pipeline Optimizer — Biomimétique

> *"La colonie de fourmis couperuses de feuilles n'a pas de chef de projet.
> Pourtant elle déplace 15% de la végétation tropicale de manière optimale."*

## Les 9 modèles biologiques → patterns CI

Read `references/patterns.md` for detailed explanations, biology, and dual GitLab/GitHub examples.

| # | Organisme | Pattern CI | Principe clé |
|---|-----------|-----------|--------------|
| 1 | **Fourmis couperuses** (Atta) | Stage Specialization + Artifact Cache | Chaque job = un rôle. Artifacts = stigmergie |
| 2 | **Myxomycète** (Physarum) | Change-Driven Path Selection + Cache Reinforcement | Ne relancer que ce qui a changé. Cache hash contenu |
| 3 | **Fourmis légionnaires** (Eciton) | Fan-out / Fan-in + Ephemeral Runners | Parallélisme maximal, runners jetables |
| 4 | **Abeilles** (Apis) | Autoscaling + Priority-based Runner Tagging | Resources proportionnelles à la charge |
| 5 | **Mycelium** | Multi-Registry Fallback + Distributed Cache | Zéro SPOF, cache waterfall |
| 6 | **Mitose** | Workflow Fission | Pipeline trop gros → diviser en workflows indépendants |
| 7 | **Système immunitaire** (VDJ) | Combinatorial Fuzzing + Property-Based Testing | Explorer l'espace des inputs, pas les cas connus |
| 8 | **Spores fongiques** | Full Combinatorial Matrix | OS × arch × version × features, fail-fast: false |
| 9 | **Tardigrade** | Chaos Engineering / Fault Injection | Injecter les pannes pour prouver la survie |

## Workflow d'analyse d'un pipeline existant

### Étape 1 — Cartographie DAG réel
Dessiner les dépendances réelles. Identifier les jobs séquentiels qui pourraient être parallèles. Calculer le chemin critique.

### Étape 2 — Audit fourmis couperuses
Chaque job fait-il UNE chose ? Les artifacts sont-ils bien définis (stigmergie) ? Y a-t-il des jobs généralistes à découper ?

### Étape 3 — Audit blob (Physarum)
Y a-t-il des jobs qui tournent sans changement pertinent ? Les clés de cache sont-elles basées sur le contenu ? Les retries sont-ils sélectifs ?

### Étape 4 — Audit légionnaires
Quels jobs lents peuvent être fan-out en parallèle ? Les runners sont-ils éphémères ? Y a-t-il des fan-in inutilement précoces ?

### Étape 5 — Audit abeilles
Les runners sont-ils dimensionnés au bon profil ? Y a-t-il du scale-to-zero ? Les jobs critiques ont-ils des runners dédiés ?

### Étape 6 — Audit mycelium
Y a-t-il des SPOF (registry, cache, runner unique) ? Le cache a-t-il un fallback waterfall ? Les pipelines multi-projets partagent-ils les artifacts coûteux ?

### Étape 7 — Audit système immunitaire
Y a-t-il du fuzzing sur les parsers, la sérialisation, l'auth ? Les tests utilisent-ils du property-based testing ? Le corpus de crashs est-il persisté entre runs ?

### Étape 8 — Audit spores
La matrice couvre-t-elle OS × arch × version × features ? Les combinaisons nightly/beta sont-elles en allow_failure ? Y a-t-il des combinaisons impossibles à exclure ?

### Étape 9 — Audit tardigrade
Y a-t-il des tests en conditions réseau dégradé ? Y a-t-il des tests sous pression ressources ? Y a-t-il des tests avec clock skew ? La dégradation gracieuse est-elle vérifiée ?

---

## Règles impératives

### Vérification des versions d'Actions GitHub

**Avant de recommander un bump de version d'une GitHub Action** (ex: `@v3` → `@v4`), vérifier que le tag cible existe réellement :

1. Les tags **majeurs flottants** (`@v4`) sont une convention, pas une obligation. Certains mainteneurs ne les créent pas.
2. **Toujours vérifier** via `gh api repos/{owner}/{repo}/git/refs/tags/{tag}` que le tag exact existe.
3. Si le tag majeur flottant n'existe pas, **épingler sur la dernière version patchée** (ex: `@v4.1.1`).
4. Ne jamais supposer qu'un tag majeur existe juste parce que des tags patchés existent.

> **Incident de référence** : bump `cosign-installer@v3` → `@v4` a cassé la CI car le tag `v4` n'existait pas (2026-03-27).

### Gotchas

Lire `../../gotchas.md` avant de produire un output pour éviter les erreurs connues.

---

## Anti-patterns à identifier

| Anti-pattern | Biologie cassée | Fix GitLab | Fix GitHub Actions |
|---|---|---|---|
| Jobs séquentiels sans dépendance réelle | Fourmi généraliste | `needs:` DAG direct | `needs:` entre jobs |
| Cache key = branche ou date | Blob sans mémoire | `key: files: [Cargo.lock]` | `key: rust-${{ hashFiles('Cargo.lock') }}` |
| Retry sur tout | Blob qui retrace les mauvais chemins | `when: [runner_system_failure]` | `nick-fields/retry` avec `retry_on: error` |
| Rebuild complet sur changement mineur | Blob sans pruning | `rules: changes:` | `on.push.paths` ou `dorny/paths-filter` |
| Runner unique pour tout | Pas de division des castes | `tags:` + autoscaler | Runner labels + larger runners |
| Registry sans fallback | Mycelium sans redondance | `cmd1 \|\| cmd2 \|\| cmd3` | idem |
| Artifacts énormes transmis partout | Porteuse qui transporte tout à tous | Artifacts scoped, `needs:` sélectif | Artifacts nommés + `download-artifact` sélectif |
| Tests non-shardés | Légionnaires sans flancs | `parallel: N` + sharding | `strategy.matrix` + sharding |
| Tests uniquement sur cas écrits à la main | Immunitaire sans VDJ | `cargo fuzz` + `proptest` | idem |
| Une seule combinaison OS/version | Spore unique sans dispersion | `parallel: matrix:` | `strategy.matrix` combinatoire |
| Pas de test en conditions dégradées | Tardigrade sédentaire | Toxiproxy + stress-ng + faketime | idem |

---

## Métriques de succès

- **Chemin critique** : < 50% du temps total théorique séquentiel
- **Cache hit rate** : > 80% sur les dépendances
- **Runner utilization** : > 70%
- **Flaky test rate** : < 1%
- **SPOF count** : 0

---

## Pipeline Scoring (15 dimensions)

Read `references/scoring.md` for detailed scoring criteria and scorecard template.

| # | Dimension | 0 | 4 |
|---|-----------|---|---|
| D1 | **DAG** | Tout séquentiel | DAG + pruning dynamique |
| D2 | **Cache** | Aucun | Cross-pipeline + GC |
| D3 | **Parallélisme** | 1 runner | Auto-scale + spot |
| D4 | **Résilience** | Pas de retry | Multi-provider + self-healing |
| D5 | **Feedback** | > 15 min | < 2 min (smoke) |
| D6 | **Pruning** | Rebuild tout | Predictive skip |
| D7 | **Artifacts** | Tout partagé | Content-addressed |
| D8 | **Security** | Aucun scan | SBOM + signing |
| D9 | **Observabilité** | Logs bruts | Anomaly detection |
| D10 | **Mitose** | 1 mega-pipeline | Event-driven mesh |
| D11 | **Coût** | Pas de mesure | FinOps optimized |
| D12 | **DX** | Config manuelle | GitOps + preview envs |
| D13 | **Fuzzing** | Aucun | Corpus persistant + regression |
| D14 | **Matrix** | 1 env | Full combinatorial + nightly |
| D15 | **Chaos** | Aucun | Chaos Monkey prod + observabilité |

**Score cible :** > 45/60 (75%) pour un projet production.

---

## Pre-Mortem Pipeline

Avant de merger un changement de pipeline, imaginer les échecs :

1. Cache empoisonné (artefact malveillant)
2. Runner compromis (secret leak)
3. Registry down (pas de fallback)
4. Job flaky retry-masqué pendant 3 mois
5. Cache miss → pipeline 45 min
6. Cross-workflow dependency cassée silencieusement
7. Diamant de dépendances DAG (C échoue silencieusement)

Pour chaque scénario : quelle mitigation existe DÉJÀ ? Si aucune → risque accepté, documentez-le.

---

## Pipeline Mutation Testing

| Mutation | Attendu | Si ça passe = bug |
|----------|---------|-------------------|
| Supprimer un `needs:` | Job tourne trop tôt | DAG mal configuré |
| Cache key `always-hit` | Cache empoisonné détecté | Pas de validation |
| Supprimer `paths:` filter | Tout rebuild (lent) | OK (conservative) |
| `continue-on-error: true` partout | Release malgré failure | Gate manquant |
| Supprimer security scan | Release sans scan | Pas de gate security |
| Doubler `timeout-minutes` | Job lent pas détecté | Pas d'alerte durée |

Si > 2 mutations passent silencieusement, le pipeline a des trous.

---

## Templates de référence

- `references/patterns.md` — Les 9 modèles biologiques détaillés avec exemples GitLab + GitHub Actions
- `references/scoring.md` — Scoring 15 dimensions détaillé + scorecard
- `references/gitlab-ci-biomimic.yml` — Pipeline GitLab complet
- `references/github-actions-biomimic.yml` — Pipeline GitHub Actions complet
- `references/runner-autoscaler.toml` — Config GitLab Runner autoscaler (abeilles)
- `references/cache-strategy.md` — Stratégies de cache avancées (blob + mycelium)

## Integration with other cli-* skills

| Skill | Relationship |
|-------|-------------|
| `cli-audit-test` | D13 couvre le drift detection dans les tests. cli-forge-pipeline couvre la **CI qui exécute** ces tests |
| `cli-audit-code` | Audite la qualité du code. cli-forge-pipeline audite la **qualité du pipeline** |
| `cli-cycle` | Appelle cli-forge-pipeline comme partie du review complet |
