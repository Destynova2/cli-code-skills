# Conflict Resolution — Decision Tree

> Sources: Anthropic C compiler build, Claude Code Agent Teams docs, AlphaCode, MetaGPT, Addy Osmani "Code Agent Orchestra"

## Principes

1. **Prevention > Detection > Resolution.** Investir 80% en prevention, 15% en detection, 5% en resolution.
2. **Le Sous-Chef gere les conflits, jamais les Commis.** Un commis qui rebase seul peut casser le travail d'un autre.
3. **3 tentatives max.** Apres 3 echecs de resolution, escalade au Chef qui decide (re-assigner, sequencer, ou fusionner manuellement).

## Arbre de decision

```
Un commis annonce "Pret !"
  │
  ▼
Le Sous-Chef tente le merge
  │
  ├─ OK (pas de conflit) ──────────── Merge + CI + "Plat envoye"
  │
  └─ CONFLIT detecte
       │
       ▼
     Quel type de conflit ?
       │
       ├─ Fichier different, meme ligne (ex: Cargo.toml dependencies)
       │    │
       │    ▼
       │  RESOLUTION AUTO : merge des deux ajouts
       │  (le Sous-Chef fait `git checkout --theirs` ou merge manuel)
       │  → Re-run CI → si OK → "Plat envoye"
       │
       ├─ Meme fichier, meme fonction (conflit semantique)
       │    │
       │    ▼
       │  Le Sous-Chef demande aux 2 commis concernes :
       │  "Quel est l'intention de votre changement sur fn X ?"
       │  → Compare les intentions
       │    │
       │    ├─ Intentions compatibles → Sous-Chef fusionne
       │    │
       │    └─ Intentions contradictoires → Escalade au Chef
       │         │
       │         ▼
       │       Le Chef decide :
       │         ├─ Garder approche A (re-assigner B)
       │         ├─ Garder approche B (re-assigner A)
       │         ├─ Fusionner (nouvelle spec)
       │         └─ Sequencer (A d'abord, puis B s'adapte)
       │
       └─ Fichier de config partage (ci.yml, Cargo.toml, package.json)
            │
            ▼
          REGLE G15 : Sequencer les merges
          1. Merger la PR la plus ancienne d'abord
          2. Rebase la suivante sur le resultat
          3. Re-run CI sur la PR rebasee
          4. Merger si CI verte
```

## Protocole du Sous-Chef (avant chaque merge)

```
1. git fetch origin
2. git log --oneline origin/{base}..{worker_branch} → lister les fichiers touches
3. Comparer avec les fichiers des PRs deja mergees dans cette session
4. SI overlap detecte :
     → Tenter merge local (sans push)
     → SI conflit :
         → Lire le diff des deux cotes
         → SI resolution triviale (ajouts non-conflictuels) : resoudre auto
         → SINON : demander aux commis concernes via SendMessage
5. Run CI localement (cargo test / npm test / etc.)
6. SI CI verte : push + "Plat envoye"
7. SI CI rouge : identifier la regression, renvoyer au commis
```

## Compteur de tentatives

| Tentative | Action |
|-----------|--------|
| 1 | Sous-Chef tente merge auto (trivial conflicts) |
| 2 | Sous-Chef demande aux commis d'expliquer leur intention, fusionne |
| 3 | Escalade au Chef : decision humaine ou re-assignation |

Apres la tentative 3, le Chef peut :
- **Re-assigner** : un seul commis prend les deux taches
- **Sequencer** : transformer les taches paralleles en dependances dans le PERT
- **Diviser** : splitter le fichier en conflit en modules separes (refactoring)

## Prevention renforcee (Phase 0)

### Cycle detection sur le PERT (Tarjan — inspire de cli-audit-tangle)

Avant de lancer les commis, verifier que le graphe de dependances est un DAG :

```
1. Construire le graphe : tache_A → tache_B si A doit finir avant B
2. Appliquer Tarjan SCC sur le graphe
3. Si SCC avec > 1 noeud → CYCLE DETECTE → ne pas lancer

Exemple de cycle interdit :
  tache_1 (auth) depend de tache_2 (database schema)
  tache_2 (database) depend de tache_1 (auth models)
  → SCC = {tache_1, tache_2} → CYCLE

Resolution :
  - Fusionner les 2 taches en 1 (meme commis)
  - OU casser la dependance (interface/mock)
  - OU sequencer explicitement (A puis B, pas parallele)
```

### Forward-only rule

Les dependances entre commis ne peuvent aller que dans UN sens :
- Commis A peut dependre du resultat de Commis B
- Mais B ne peut PAS dependre de A en retour
- Si bi-directionnel → fusionner en un seul commis

### God-commis detection (Fourmis de feu)

Un commis qui touche trop de fichiers = fourmi qui agrippe 12 voisines = SPOF.

```
Pour chaque commis, compter :
  files_touched = nombre de fichiers dans sa tache
  total_files = nombre total de fichiers touches par tous les commis

Si files_touched / total_files > 0.5 → GOD-COMMIS
  → Splitter la tache en 2 commis
  → OU re-assigner certains fichiers a d'autres commis

Si un commis a des W/W avec > 2 autres commis → HUB
  → Sequencer ses merges en dernier
  → OU le transformer en "commis de base" dont les autres dependent
```

### Matrice de couplage

Avant le coup de feu, le Chef construit une matrice :

```
          fichier_a  fichier_b  fichier_c  Cargo.toml
tache_1      W          R          -          W
tache_2      -          W          W          W
tache_3      R          -          W          R
```

- **W/W sur la meme colonne** = conflit garanti → assigner au meme commis OU sequencer
- **R/W** = risque faible, acceptable en parallele
- **R/R** = aucun risque

### Fichiers "hot" (toujours en conflit)

Certains fichiers sont touches par presque toutes les taches :
- `Cargo.toml` / `package.json` (ajout de deps)
- `ci.yml` / `.github/workflows/` (ajout de jobs)
- `mod.rs` / `index.ts` (re-exports)
- `CHANGELOG.md`

**Regle** : ces fichiers sont geres par le Sous-Chef APRES tous les merges. Les commis ne les touchent pas directement — ils listent leurs besoins dans shared-state.md ("Besoin: ajouter dep X dans Cargo.toml") et le Sous-Chef les applique en une passe finale.

## File locking via shared-state.md

Inspiree du pattern d'Anthropic (C compiler build) :

```markdown
## Verrous actifs

| Fichier | Commis | Depuis | Raison |
|---------|--------|--------|--------|
| src/auth/mod.rs | commis-1 | 14:32 | refactoring auth flow |
```

**Regles :**
1. Un commis qui veut modifier un fichier verifie d'abord les verrous
2. Si verrouille par un autre → SendMessage au Sous-Chef : "Besoin de modifier {fichier}, verrouille par {commis}"
3. Le Sous-Chef decide : attendre, re-assigner, ou autoriser (si zones differentes dans le fichier)
4. Le commis libere le verrou en faisant "Pret !"
