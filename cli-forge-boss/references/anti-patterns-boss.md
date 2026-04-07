# Anti-Patterns Brigade — Detection et Resolution

> **When to read:** Pendant l'execution (Phase 2+), pour identifier et corriger les dysfonctionnements de la brigade.

---

## Anti-patterns nommes

| # | Nom | Detection | Severite | Resolution |
|---|-----|-----------|----------|------------|
| B1 | **Ping-Pong Rejection** | Meme commis × meme sous-chef → reject le meme diff 3+ fois | Critique | Patch Bankruptcy : re-assigner ou redefinir la spec |
| B2 | **Hot File Thrashing** | 3+ commis editent le meme fichier dans le meme sprint | Majeur | Sequencer les merges (G15) ; marquer le fichier en zone sensible (3/3) |
| B3 | **Ghost Commis** | Commis assigne mais aucun commit/message depuis > 30min | Majeur | Timeout, re-assigner, diagnostiquer (NFD : peut-il communiquer ?) |
| B4 | **Gate Consensus Flip** | Sous-chef approuve un diff puis le rejette au round suivant (ou inverse) | Majeur | Auditer le sous-chef — ses regles sont incoherentes |
| B5 | **God Commis** | Commis touche > 50% des fichiers du sprint OU > 5 taches independantes | Majeur | Splitter, re-assigner, ajouter un commis |
| B6 | **Speculative Task** | Commis commence avant que ses tokens de dependance soient deposes | Mineur | Rappeler la regle stigmergie ; si intentionnel, marquer comme EXPLORATION |
| B7 | **Dead Branch** | Branche d'un commis existe depuis > 2 sprints sans merge | Mineur | Archiver, nettoyer, notifier |
| B8 | **Feature Envy Inter-Commis** | Commis A depend de > 3 outputs de Commis B | Majeur | Co-assigner au meme commis OU pre-calculer la dependance partagee |
| B9 | **Sous-Chef Bypass** | Commis merge directement sans passer par le vote | Critique | Bloquer les permissions ; le commis ne doit JAMAIS avoir push sur la branche principale |
| B10 | **Over-Staffing** | 5+ commis pour < 3 taches independantes | Mineur | Reduire la brigade (mitosis tier S/M) |

---

## Detection heuristique

### B1 — Ping-Pong Rejection
```
Pour chaque paire (commis, sous-chef) :
  count = nombre de rejets consecutifs sur le meme fichier/diff
  IF count >= 3 → PATCH BANKRUPTCY
```

### B3 — Ghost Commis
```
Pour chaque commis actif :
  last_activity = max(dernier commit, dernier SendMessage, dernier edit shared-state)
  IF now - last_activity > 30min → GHOST
  Diagnostic NFD :
    1. Le commis peut-il lire shared-state.md ? (test read)
    2. Le commis peut-il ecrire dans son worktree ? (test write)
    3. Le commis peut-il envoyer un SendMessage ? (test comm)
    SI echec → probleme de communication, pas de performance
```

### B5 — God Commis
```
Pour chaque commis :
  files_ratio = files_touched(commis) / total_files_sprint
  task_count = nombre de taches assignees
  IF files_ratio > 0.5 OR task_count > 5 → GOD COMMIS
```

### B8 — Feature Envy Inter-Commis
```
Pour chaque paire (commis_A, commis_B) :
  deps = nombre de tokens de A consommes par B
  IF deps > 3 → FEATURE ENVY
  → Co-assigner ou pre-calculer
```
