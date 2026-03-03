# Doc Forge Advanced — Méthodes pour approcher 100% (et dépasser)

## La philosophie du "120%"

100% = toute l'information nécessaire est documentée.
120% = la documentation **ajoute de la valeur** au-delà de l'information brute : elle anticipe les questions, elle connecte les concepts, elle accélère la compréhension.

Le delta entre 100% et 120%, c'est :
- Des exemples qui couvrent les edge cases, pas juste le happy path
- Des ADRs qui documentent les alternatives **rejetées** (pas juste le choix final)
- Des liens entre concepts que le lecteur n'aurait pas fait seul
- Des warnings sur les pièges que seul un expert connaît

---

## Partie 1 : Le Graphe de Concepts (Concept Dependency Graph)

### Principe

Un projet n'est pas une liste plate d'items à documenter. C'est un **graphe orienté** de concepts où chaque concept dépend d'autres concepts pour être compris.

```
Concept A ──→ Concept B ──→ Concept D
    │                           ↑
    └──→ Concept C ─────────────┘
```

Si tu documentes D sans documenter B et C, le lecteur est perdu. Si tu documentes B sans A, pareil.

### Construction du graphe

**Étape 1 : Extraire les concepts**

Pour chaque fichier source, extraire :
- Types publics (structs, enums, traits, interfaces, classes)
- Fonctions publiques
- Constantes de configuration
- Modules/packages

Chaque item = un **nœud** dans le graphe.

**Étape 2 : Extraire les dépendances**

Pour chaque nœud, identifier :
- Quels types il utilise en paramètre ou retour → **dépendance directe**
- Quels concepts il faut comprendre pour l'utiliser → **dépendance cognitive**
- Quels modules il importe → **dépendance structurelle**

Chaque dépendance = une **arête** dans le graphe.

**Étape 3 : Calculer les métriques du graphe**

```
Couverture de nœud = Nœuds documentés / Nœuds totaux

Couverture d'arête = Liens documentés (cross-refs) / Liens existants

Couverture pondérée = Σ(doc_score(n) × importance(n)) / Σ importance(n)
```

Où `importance(n)` est calculée par :

```
importance(n) = in_degree(n) + out_degree(n) + is_public(n) × 3

Intuition : un type utilisé partout (haut in_degree) est plus
important à documenter qu'un helper interne utilisé une fois.
```

### Détection des trous (Gap Analysis)

```
gap_score(n) = importance(n) × (1 - doc_score(n))
```

Trier par `gap_score` décroissant → les items en haut de la liste sont les plus urgents à documenter. C'est la **file de priorité de documentation**.

### Détection des orphelins

Un **orphelin** est un concept documenté dont les dépendances ne le sont pas :

```
orphan(n) = doc_score(n) > 0.5 AND
            ∃ dep ∈ dependencies(n) : doc_score(dep) < 0.25
```

Un orphelin crée une **falaise cognitive** : le lecteur comprend le nœud mais pas ses prérequis.

### Couverture de chemin (Path Coverage)

```
Un "chemin d'apprentissage" P est une séquence ordonnée :
P = [concept_1, concept_2, ..., concept_k]

où chaque concept_i+1 dépend de concept_i.

Path coverage = Chemins entièrement documentés / Chemins totaux
```

Un chemin est "entièrement documenté" si TOUS ses nœuds ont `doc_score ≥ 0.75`.

**Objectif** : Path coverage ≥ 95% sur les chemins partant des entry points publics.

---

## Partie 2 : Entropie Documentaire (Documentation Need Score)

### Principe

L'entropie de Shannon mesure l'incertitude. Appliquée à la doc :
- **Haute entropie** = le lecteur ne peut pas prédire le comportement → documentation nécessaire
- **Basse entropie** = le comportement est évident par le nom/type → pas besoin de documenter

### Formule : Documentation Need Score (DNS)

```
DNS(item) = H(behavior) - I(name, types, context)

Où :
  H(behavior) = entropie du comportement de l'item
                (combien de comportements possibles ?)
  I(name, types, context) = information mutuelle entre
                le nom/signature et le comportement réel
```

**Intuition** : si le nom `get_user_by_id(id: UserId) -> Option<User>` dit déjà tout sur le comportement, le DNS est bas. Si `process(data: &[u8]) -> Result<Vec<Output>>` est ambigu, le DNS est haut.

### Heuristique pratique (sans calcul formel)

| Signal | DNS | Action |
|--------|-----|--------|
| Nom descriptif + types forts + pas d'effets de bord | Bas | Summary line suffit |
| Nom générique (`process`, `handle`, `run`) | Haut | Doc complète avec exemples |
| Retourne `Result` ou `Option` | +1 | Documenter les cas d'erreur |
| Prend des `&[u8]`, `Any`, `impl Trait` | +1 | Documenter les invariants |
| A des effets de bord (IO, mutation, réseau) | +2 | Documenter le comportement complet |
| Est `unsafe` | +3 | Documentation SAFETY obligatoire |
| A plus de 3 paramètres | +1 | Documenter les interactions entre params |

### Score d'entropie d'un module

```
Module_entropy = Σ DNS(item_i) pour item_i ∈ module

Module_doc_quality = Σ min(DNS(item_i), doc_depth(item_i)) / Module_entropy
```

Un module avec `Module_doc_quality ≥ 1.0` est documenté à hauteur de sa complexité. C'est ça le "100%".

Un module avec `Module_doc_quality ≥ 1.2` dépasse les attentes — c'est le "120%" :
exemples supplémentaires, edge cases documentés, liens vers l'architecture, warnings sur les pièges.

---

## Partie 3 : Élimination systématique des erreurs

### Mutation Testing pour la documentation

Inspiré du mutation testing en software : on "casse" la doc et on vérifie si quelqu'un s'en rendrait compte.

| Mutation | Test | Si ça passe = problème |
|----------|------|------------------------|
| Supprimer un paragraphe | Le lecteur peut-il encore accomplir la tâche ? | La doc était redondante (OK) ou le test est mauvais |
| Inverser un paramètre | Le lecteur va-t-il se tromper ? | Si non → la doc ne clarifie rien d'utile |
| Changer un type de retour dans la doc | Le lecteur va-t-il remarquer ? | Si non → la doc n'est pas lue/utile |
| Supprimer un exemple | Le concept reste-t-il clair ? | Si oui → l'exemple n'ajoutait rien |

### Cross-validation Code / Doc

**Pour chaque item public documenté :**

```
1. Parser le doc comment → extraire : params, return type, errors, panics
2. Parser le code → extraire : signature réelle, ? operators, panic! calls, unsafe blocks
3. Comparer :
   - Params dans la doc ⊆ params dans le code ? (complétude)
   - Params dans le code ⊆ params dans la doc ? (exhaustivité)
   - Errors documentés ⊇ error types retournés ? (couverture d'erreur)
   - Si unsafe dans le code → SAFETY dans la doc ? (sécurité)
```

**Formule de cohérence :**

```
coherence(item) = |documented_facts ∩ actual_facts| / |documented_facts ∪ actual_facts|

Où :
  documented_facts = {params, returns, errors, panics, safety} extraits de la doc
  actual_facts = {params, returns, errors, panics, safety} extraits du code

coherence = 1.0 → doc et code sont en sync parfait
coherence < 0.8 → flag pour review
coherence < 0.5 → critical: la doc est probablement stale
```

### Lint automatique (intégration CI)

| Langage | Outil | Ce qu'il catch |
|---------|-------|----------------|
| Rust | `clippy` (missing_docs, missing_errors_doc) | Couverture + sections |
| Rust | `cargo doc --check` | Liens cassés, exemples qui compilent pas |
| Rust | `doc-coverage` (crate) | % de couverture exact |
| TypeScript | `typedoc` + plugins | Couverture + format |
| Python | `interrogate` | % de couverture docstring |
| Python | `darglint` | Cohérence params doc / signature |
| Go | `golint` / `revive` | Exports sans doc |
| Tout | `vale` | Style, consistance, jargon |
| Tout | `cspell` | Typos dans la doc |

---

## Partie 4 : Score DCI Étendu (DCI+)

### Au-delà du DCI de base

Le DCI de base mesure "est-ce que ça existe ?". Le DCI+ mesure "est-ce que c'est **bon** ?"

```
DCI+ = DCI_base × Quality_multiplier × Coverage_multiplier

Quality_multiplier = (1 + bonus_examples + bonus_crossrefs + bonus_adrs) / 4

  bonus_examples = 0.0 à 1.0 (proportion d'items avec exemples testés)
  bonus_crossrefs = 0.0 à 1.0 (proportion de refs intra-doc fonctionnelles)
  bonus_adrs = 0.0 à 1.0 (proportion de décisions architecturales documentées)

Coverage_multiplier = path_coverage × node_coverage

  path_coverage = chemins d'apprentissage entièrement documentés
  node_coverage = nœuds du concept graph documentés
```

### Grille DCI+

| DCI+ | Grade | Signification |
|------|-------|---------------|
| > 12.0 | S | Exceptionnel — la doc est un avantage compétitif (le "120%") |
| 10.0–12.0 | A+ | Exemplaire — rien à redire |
| 9.0–10.0 | A | Excellent — quelques bonus manquants |
| 7.0–8.9 | B | Solide — le minimum est couvert |
| < 7.0 | C ou moins | Voir DCI de base |

### Comment atteindre S (> 12.0)

Checklist des bonus :

- [ ] 100% des items publics ont un doc comment non-tautologique
- [ ] 100% des Result-returning fns ont `# Errors`
- [ ] 95%+ des items publics ont au moins un exemple
- [ ] Tous les exemples compilent/passent
- [ ] Tous les types mentionnés sont liés (intra-doc links)
- [ ] Architecture documentée avec au moins 3 ADRs
- [ ] Getting-started testé par un non-expert en < 10 min
- [ ] llms.txt + AGENTS.md à jour
- [ ] Glossaire des termes métier
- [ ] Changelog à jour
- [ ] Zéro doc stale (cohérence ≥ 0.95)
- [ ] Transitions narratives entre toutes les pages
- [ ] CI lint (vale + language-specific linter) qui passe
