# Exploration Parallele — Best-of-N & Benchmarking

> Sources: AlphaCodium (Qodo), AlphaCode (DeepMind), Cursor Shadow Workspace, Claude Code Competing Hypotheses, ICLR 2025 MAD analysis

## Quand utiliser ce pattern

| Situation | Pattern standard (1 commis/tache) | Exploration parallele (N commis/tache) |
|-----------|------------------------------------|---------------------------------------|
| Tache bien definie, une seule approche evidente | ✅ | ❌ overkill |
| Choix d'architecture (2-3 options valides) | ❌ risque d'ancrage | ✅ |
| Bug complexe, cause racine inconnue | ❌ biais de confirmation | ✅ (hypotheses concurrentes) |
| Optimisation performance (algorithme, structure) | ❌ | ✅ (bench obligatoire) |
| Refactoring avec plusieurs decoupages possibles | ❌ | ✅ (comparer les resultats) |

**Regle : ne PAS utiliser pour plus de 30% des taches d'un sprint.** L'exploration parallele coute 2-3x plus de tokens. Reserver aux decisions a fort impact.

## Pattern 1 — Hypotheses Concurrentes (Debug)

Pour les bugs complexes ou la cause racine n'est pas evidente.

```
Le Chef identifie un bug
  │
  ▼
Formuler 2-3 hypotheses distinctes :
  H1: "Le bug vient du cache invalide"
  H2: "Le bug vient d'une race condition"
  H3: "Le bug vient d'un parsing incorrect"
  │
  ▼
Spawner 1 commis par hypothese (worktree separe chacun)
  - Chaque commis recoit UNE hypothese
  - Mission : prouver OU refuter l'hypothese
  - Produire : evidence (logs, tests, reproducing scenario)
  │
  ▼
Les commis envoient leurs conclusions au Sous-Chef
  │
  ▼
Le Sous-Chef compare :
  - Quelle hypothese a des preuves reproductibles ?
  - Quelle hypothese explique TOUS les symptomes ?
  │
  ▼
Le Chef valide et assigne le fix au commis gagnant
```

**Anti-pattern : NE PAS faire debattre les commis entre eux.** L'ICLR 2025 montre que le debat multi-agent bat rarement un agent seul + chain-of-thought. Le Sous-Chef juge, les commis ne se parlent pas.

## Pattern 2 — Approches Concurrentes (Architecture / Refactoring)

Pour les choix avec 2-3 solutions valides.

```
Le Chef definit le probleme + les contraintes
  │
  ▼
Assigner a 2-3 commis :
  Commis A : "Implemente avec l'approche X (ex: trait objects)"
  Commis B : "Implemente avec l'approche Y (ex: enum dispatch)"
  Commis C : "Implemente avec l'approche Z (ex: generics)"
  │
  ▼
Chaque commis produit :
  1. Implementation fonctionnelle (compile + tests passent)
  2. Benchmark (si applicable)
  3. Note : complexite, lignes de code, maintenabilite
  │
  ▼
Le Sous-Chef execute la grille de comparaison (voir ci-dessous)
  │
  ▼
Le Chef choisit + les branches perdantes sont supprimees
```

### Grille de comparaison

Le Sous-Chef remplit cette grille pour chaque approche :

| Critere | Poids | Approche A | Approche B | Approche C |
|---------|-------|-----------|-----------|-----------|
| Tests passent | 10 | ✅/❌ | ✅/❌ | ✅/❌ |
| CQI (/cli-audit-code) | 5 | score | score | score |
| Lignes de code | 3 | N | N | N |
| Complexite cyclomatique | 4 | N | N | N |
| Performance (bench) | 5 (si applicable) | ms/op | ms/op | ms/op |
| Couplage (/cli-audit-tangle) | 4 | score | score | score |
| Facilite d'extension future | 3 | 1-5 | 1-5 | 1-5 |

**Score** = somme(poids × note normalisee). L'approche avec le meilleur score gagne.

**Filtre eliminatoire** : si tests ne passent pas → elimine immediatement (pas de score).

## Pattern 3 — Flow AlphaCodium (Generation + Selection iterative)

Pour les problemes algorithmiques ou les implementations critiques. Plus leger que le full parallel (2-3 variantes, pas N).

```
Phase 1 — Reflexion
  Le commis analyse le probleme :
  - Objectifs, contraintes, cas limites
  - Pourquoi chaque test attend ce resultat
  │
Phase 2 — Generer 2-3 solutions
  Le commis produit 2-3 variantes DANS LE MEME WORKTREE :
  - solution_a.rs, solution_b.rs, solution_c.rs
  │
Phase 3 — Classer
  Le commis execute les tests sur chaque variante
  Elimine celles qui echouent
  Garde la plus simple parmi celles qui passent
  │
Phase 4 — Generer des tests adversariaux
  Le commis invente 6-8 cas limites supplementaires
  Les execute sur la solution retenue
  │
Phase 5 — Iterer
  Si un test adversarial echoue → corriger et revenir a Phase 4
  Si 3 iterations sans convergence → escalade au Chef
  │
Phase 6 — Livrer
  Supprimer les variantes perdantes
  Commit uniquement la solution gagnante
```

**Avantage** : ne coute qu'un seul commis (pas de duplication de worktree).

## Integration dans le boss

### Dans le PERT

Marquer les taches en exploration parallele :

```
┌──────────────────────────────────┐
│  T3: Refactoring auth module     │
│  MODE: EXPLORATION (2 commis)    │
│  Approche A: trait objects        │
│  Approche B: enum dispatch        │
│  GATE: comparaison + selection    │
└──────────────────────────────────┘
```

### Dans shared-state.md

Ajouter une section :

```markdown
## Explorations en cours

| Tache | Commis | Approche | Status | Score |
|-------|--------|----------|--------|-------|
| T3 | commis-1 | trait objects | En cours | - |
| T3 | commis-2 | enum dispatch | En cours | - |
```

### Dans le prompt du Chef

```
Pour les taches marquees EXPLORATION :
1. Spawner N commis avec chacun une approche differente
2. Attendre que tous aient fini (ou timeout 30min)
3. Demander au Sous-Chef d'executer la grille de comparaison
4. Choisir l'approche gagnante
5. Supprimer les worktrees/branches perdantes
6. Continuer le PERT avec la branche gagnante
```

### Quand le Chef decide d'utiliser l'exploration

Le Chef propose l'exploration parallele si :
- Le user a explicitement demande "compare les approches"
- La tache a un scope > S dans la mitosis
- Il y a un choix d'architecture non-trivial (2+ patterns applicables)
- Le user n'a pas specifie d'approche precise

Le Chef demande confirmation au user avant de lancer (cout tokens x2-3).
