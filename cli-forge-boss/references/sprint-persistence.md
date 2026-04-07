# Sprint Persistence — Checkpoint, Resume, Rewind

> **When to read:** When the user wants to pause/resume a sprint, rewind to a previous state, or start fresh while keeping learnings.

---

## Concept (inspire de jj operation log)

Chaque sprint laisse des **checkpoints** — des snapshots de l'etat complet du projet a des moments cles. Le Chef peut naviguer entre ces checkpoints comme `jj undo` ou `git reset`.

```
Sprint S1          Sprint S2          Sprint S3
  │                  │                  │
  ├─ checkpoint-0    ├─ checkpoint-0    ├─ checkpoint-0
  │  (avant sprint)  │  (avant sprint)  │  (avant sprint)
  │                  │                  │
  ├─ checkpoint-1    ├─ checkpoint-1    │  ← on est ici
  │  (apres Wave 1)  │  (apres 2 plats) │
  │                  │                  │
  ├─ checkpoint-2    ├─ checkpoint-2    │
  │  (apres Wave 2)  │  (fin sprint)    │
  │                  │                  │
  └─ rapport-S1      └─ rapport-S2      │
```

Le user peut :
- **Pause** : arreter le sprint, sauver l'etat, reprendre plus tard
- **Resume** : relancer depuis le dernier checkpoint
- **Rewind** : revenir a un checkpoint precedent (comme `jj undo`)
- **Fresh** : repartir de zero mais garder les gotchas et anti-patterns appris

---

## Checkpoints (sauvegardes automatiques)

### Quand sauver un checkpoint

| Evenement | Tag git | Contenu sauve |
|-----------|---------|---------------|
| Debut du sprint (Phase 0 complete) | `sprint/{id}/start` | shared-state.md, PERT, matrice couplage |
| Apres chaque plat merge | `sprint/{id}/plat-{n}` | shared-state.md, diff cumule, scores |
| Avant shutdown | `sprint/{id}/pre-shutdown` | etat complet avant rapport |
| Fin du sprint | `sprint/{id}/done` | rapport final, backlog, zones sensibles |
| Pause utilisateur | `sprint/{id}/pause` | etat au moment de la pause |

### Comment sauver

```bash
# Le Sous-Chef Merge execute apres chaque plat merge :
git tag "sprint/${SPRINT_ID}/plat-${PLAT_NUMBER}" HEAD
cp .claude/shared-state.md .claude/sprint-history/${SPRINT_ID}/checkpoint-${PLAT_NUMBER}.md
echo "$(date -Is) checkpoint-${PLAT_NUMBER} score=${SCORE}" >> .claude/sprint-history/${SPRINT_ID}/log.txt
```

### Structure sur disque

```
{project}/.claude/sprint-history/
├── S1/
│   ├── checkpoint-0.md     (shared-state au debut)
│   ├── checkpoint-1.md     (apres plat 1)
│   ├── checkpoint-2.md     (apres plat 2)
│   ├── rapport.md          (rapport final)
│   ├── log.txt             (chronologie : timestamps, scores, events)
│   └── gotchas-learned.md  (anti-patterns decouverts pendant S1)
├── S2/
│   ├── ...
└── current -> S3           (symlink vers le sprint en cours)
```

---

## Pause (arreter proprement)

L'utilisateur tape `pause` ou `Ctrl+C` ou ferme tmux.

Le Chef detecte l'arret et sauve :

```
1. Tag git : sprint/{id}/pause
2. Copie shared-state.md → sprint-history/{id}/checkpoint-pause.md
3. Log : "PAUSED at plat {n}/{total}, score {x}/10"
4. Ecrire dans shared-state.md section "Statut" : "PAUSED"
5. Lister les taches restantes dans section "Backlog"
```

**Le tmuxinator peut etre arrete proprement :** `tmuxinator stop {session}`

---

## Resume (reprendre la ou on s'est arrete)

L'utilisateur relance `/cli-forge-boss` sur le meme projet.

Le Chef detecte un sprint en pause :

```
1. Lire .claude/sprint-history/current/checkpoint-pause.md
2. Lire le log.txt pour connaitre la progression
3. Reconstruire le PERT avec uniquement les plats restants
4. Verifier l'etat git : branches des commis encore presentes ?
   - OUI → re-assigner les commis a leurs branches existantes
   - NON → re-creer les worktrees
5. Reprendre le sprint au plat suivant
```

**Presentation au user :**
```
Sprint S3 detecte en pause.
  Plats termines : 2/5 (auth-refactor, api-endpoints)
  Plats restants : 3 (db-migration, tests-e2e, docs-update)
  Score actuel : 7.2/10
  Derniere activite : il y a 3h

  [1] Reprendre (3 plats restants)
  [2] Rewind au checkpoint precedent
  [3] Repartir de zero (garder les gotchas)
  [4] Abandonner le sprint
```

---

## Rewind (revenir en arriere)

Comme `jj undo` — revenir a un etat precedent.

```
1. Lister les checkpoints disponibles :
   sprint/S3/start          (il y a 4h, score 6.8)
   sprint/S3/plat-1         (il y a 3h, score 7.1) ← auth-refactor
   sprint/S3/plat-2         (il y a 2h, score 7.2) ← api-endpoints
   sprint/S3/pause          (il y a 1h, score 7.2)

2. L'utilisateur choisit un checkpoint

3. Restaurer :
   git reset --hard sprint/S3/plat-1
   cp .claude/sprint-history/S3/checkpoint-1.md .claude/shared-state.md
   # Les plats 2+ sont annules — les branches des commis existent encore
   # mais leurs merges sont revert

4. Reprendre depuis ce point (les taches 2+ sont re-ajoutees au PERT)
```

**Cas d'usage :**
- Un plat merge a casse la CI → rewind avant ce plat, re-assigner a un autre commis
- Le Chef a fait un mauvais decoupage → rewind au debut, re-planifier

---

## Fresh restart (repartir de zero)

Quand le sprint est trop casse pour etre repare.

```
1. Sauver les learnings :
   - Copier gotchas-learned.md vers le prochain sprint
   - Copier la liste des zones sensibles (mises a jour pendant le sprint)
   - Copier les anti-patterns detectes (B1-B10)

2. Reset :
   git reset --hard sprint/S3/start  (ou main)
   Supprimer les branches des commis
   Supprimer les worktrees

3. Lancer un nouveau sprint S4 :
   - Le Chef lit gotchas-learned de S3
   - Les zones sensibles sont deja mises a jour
   - Le PERT est reconstruit from scratch
```

---

## Sprint history (navigation)

Le Chef peut consulter l'historique des sprints :

```
/cli-forge-boss --history

Sprint History — {project}

| Sprint | Date | Plats | Score | Status | Duree |
|--------|------|-------|-------|--------|-------|
| S1 | 2026-03-15 | 3/3 | 7.5 | DONE | 45min |
| S2 | 2026-03-22 | 4/5 | 8.1 | DONE | 1h20 |
| S3 | 2026-04-07 | 2/5 | 7.2 | PAUSED | 2h |

Checkpoints S3 : start, plat-1, plat-2, pause
Gotchas appris : 2 (Hot File Thrashing on ci.yml, Ghost Commis timeout)
```

---

## Integration avec jj et git

Le systeme fonctionne avec les deux :

| Operation | git | jj |
|-----------|-----|-----|
| Checkpoint | `git tag sprint/{id}/...` | `jj bookmark set sprint-{id}-...` |
| Rewind | `git reset --hard {tag}` | `jj undo --to {operation}` |
| Resume | Lire tag + restaurer shared-state | `jj restore --from {bookmark}` |
| Branches commis | `git worktree add` | `jj workspace add` |
| History | `git tag -l "sprint/*"` | `jj op log` |

jj est natif pour ce type d'operations (operation log, undo, workspace). git necessite des tags et des worktrees manuels.

---

## Regles de securite

1. **Jamais de force push** sur les tags de checkpoint — ils sont immutables
2. **Rewind = reset local uniquement** — ne touche pas le remote tant que le user n'approuve pas
3. **Fresh restart garde les learnings** — les gotchas et zones sensibles survivent
4. **Checkpoint automatique = pas d'opt-in** — sauve apres chaque plat merge, toujours
5. **Max 10 sprints en historique** — les plus anciens sont archives (tag reste, fichiers supprimes)
