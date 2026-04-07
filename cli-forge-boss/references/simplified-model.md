# Modele Simplifie — Stigmergie, Apoptose, Reparation ADN

> **When to read:** Phase 0 — pour decider quel modele utiliser (simplifie ou brigade complete).

---

## Principe : la nature ne coordonne pas, elle laisse emerger

Le modele Brigade de Cuisine (Chef + 3 Sous-Chefs + Merge + N Commis) est justifie pour les projets reglementes (XL). Pour les tiers S/M/L, la nature offre un modele plus simple : **pas de coordinateur central, les agents s'auto-organisent via l'environnement partage.**

```
Brigade complete (XL) :           Stigmergie (S/M/L) :
  Chef coordonne                    shared-state.md coordonne
  3 Sous-Chefs votent               1 Sous-Chef (3 lenses) revoit
  1 Sous-Chef Merge                 Commis mergent eux-memes
  N Commis executent                N Commis s'auto-assignent

  6+ agents                         2+ agents (1 sous-chef + N commis)
  Messages inter-agents             Lecture/ecriture shared-state
  Protocole de vote                 Decision interne du sous-chef
  Chef = SPOF                       Pas de SPOF
```

---

## Decision : quel modele utiliser

| Tier | Modele | Agents | Raison |
|------|--------|--------|--------|
| **S** (1-2 taches) | Pas de brigade | 1 commis seul | Un seul cuisinier n'a pas besoin de chef |
| **M** (3-4 taches) | **Stigmergie** | 1 sous-chef + N commis | Auto-organisation suffit |
| **L** (5+ taches) | **Stigmergie** | 1 sous-chef + N commis | Meme modele, plus de commis |
| **XL** (reglemente) | **Brigade complete** | Chef + 3×3 sous-chefs + N commis | Audit trail, conformite, tracabilite |

---

## Les 5 patterns du modele simplifie

### 1. Boids — 3 regles pour les commis (remplace le Chef runtime)

Chaque commis suit 3 regles. Pas de Chef qui assigne ou surveille.

```
SEPARATION : ne touche pas un fichier verrouille par un autre commis
             (verifier shared-state.md section "Verrous")

ALIGNEMENT : suis les memes conventions — commit style (cli-git-conventional),
             langue du projet, branch naming

COHESION   : converge vers l'objectif du sprint
             (verifier shared-state.md section "PERT" pour la progression)
```

Le Chef n'existe plus au runtime. Il genere les fichiers a Phase 0, puis disparait. Les commis se coordonnent via shared-state.md.

### 2. Quorum Sensing — 1 Sous-Chef, 3 lenses (remplace 3 Sous-Chefs votants)

Un seul agent de review qui applique 3 lenses sequentiellement. Pas de vote, pas de rounds de resolution, pas de consensus inter-agents.

```
Sous-Chef recoit un diff :

  Lens 1 — SCOPE : le changement est dans le perimetre de la tache ?
    SI hors scope → DENY("hors perimetre : touche {fichier} non assigne")

  Lens 2 — SECURITE : secrets exposes ? deps non-verifiees ? eval() ?
    SI risque secu → zone sensible : ESCALATE au patron
    SI risque secu → zone normale : DENY("secret detecte dans {fichier}:L{line}")

  Lens 3 — QUALITE : tests passent ? code propre ? pas de regression ?
    SI qualite basse → DENY("test manquant pour {fonction}")

  3 lenses passent → APPROVE
```

**Quand garder 3 sous-chefs separes :**
- Tier XL (reglemente)
- Diff > 200 lignes
- Zone sensible (3/3 unanimite requise)
- L'utilisateur le demande explicitement

### 3. Reparation ADN — quality gates adaptatifs (remplace les gates systematiques)

Les gates ne sont pas les memes pour tous les diffs. Comme la cellule : proofreading constant, mais checkpoint seulement quand il y a un probleme.

```
diff_size < 50 lignes ET zone normale :
  → PROOFREADING : le commis auto-verifie (test + lint)
  → Sous-Chef : check rapide (30s, 1 lens seulement — qualite)
  → Pas d'audit skill

diff_size 50-200 lignes OU zone hot :
  → MISMATCH REPAIR : le sous-chef lance /cli-audit-code
  → 1 skill, pas 4

diff_size > 200 lignes OU zone sensible :
  → CHECKPOINT : le sous-chef lance la batterie complete
  → /cli-audit-code + /cli-audit-shell (si .sh) + /cli-audit-drift (si CONTRACTS.md)
```

**Resultat :** 60-70% des merges (petits diffs en zone normale) ne declenchent aucun audit skill. Le sous-chef les valide en 30 secondes.

### 4. Reaction-Diffusion — pool de taches auto-organise (remplace PERT monitoring)

Le PERT est calcule a Phase 0 et ecrit dans shared-state.md. Ensuite les commis se servent dans le pool par score de readiness.

```markdown
## Pool de taches

| Tache | Readiness | Activateurs | Inhibiteurs | Commis |
|-------|-----------|-------------|-------------|--------|
| test-coverage | 1.0 | aucun requis | aucun | - |
| auth-refactor | 0.9 | schema-ready (oui) | aucun | commis-1 |
| api-endpoints | 0.3 | auth-done (non) | auth/mod.rs verrouille | - |
| docs-update | 0.1 | api-done (non), tests-done (non) | aucun | - |
```

**Regles :**
1. Un commis libre choisit la tache avec le readiness le plus haut
2. Quand un commis termine, ses tokens mettent a jour le readiness des taches dependantes
3. Pas besoin de Chef pour assigner ou rebalancer — le pool s'auto-organise
4. Si 2 commis veulent la meme tache → premier arrive, premier servi (verrou dans shared-state)

### 5. Apoptose — auto-terminaison des commis en echec (remplace Patch Bankruptcy + Divergence Detection)

Un commis qui echoue se termine proprement, sans intervention du Chef.

```
SI meme_diff_rejete >= 2 :
  → APOPTOSE (mauvaise approche)
  → git revert vers last-good-state
  → liberer tous les verrous
  → ecrire dans shared-state : "TACHE {x} : APOPTOSE par commis-{n}, raison: {motif}"
  → la tache retourne au pool avec readiness = 1.0
  → un autre commis la prendra avec une approche differente

SI idle > 30min sans progres :
  → APOPTOSE (coince)
  → meme procedure

SI regression detectee (fichier qui passait les tests ne passe plus) :
  → git revert du dernier edit
  → retenter 1 fois
  → si encore en echec → APOPTOSE
```

**Comme Kubernetes :** le pod (commis) est jetable. Le workload (tache) est ce qui compte. On ne debug pas un commis en echec, on le remplace.

---

## Architecture runtime du modele simplifie

```
┌─────────────────────────────────────────────┐
│           shared-state.md                    │
│  (le seul medium de coordination)            │
│                                              │
│  Sections :                                  │
│  - Pool de taches (readiness scores)         │
│  - Verrous actifs                            │
│  - Tokens de completion                      │
│  - Zones sensibles                           │
│  - Status des commis (ACTIF/APOPTOSE)        │
│  - Backlog pour le prochain sprint           │
└─────────┬───────────┬───────────┬────────────┘
          │           │           │
     ┌────▼────┐ ┌────▼────┐ ┌───▼─────┐
     │Commis 1 │ │Commis 2 │ │Commis N │
     │         │ │         │ │         │
     │ Boids:  │ │ Boids:  │ │ Boids:  │
     │ separ.  │ │ separ.  │ │ separ.  │
     │ align.  │ │ align.  │ │ align.  │
     │ cohes.  │ │ cohes.  │ │ cohes.  │
     │         │ │         │ │         │
     │ Apopt:  │ │ Apopt:  │ │ Apopt:  │
     │ 2 rejet │ │ 2 rejet │ │ 2 rejet │
     │ = mort  │ │ = mort  │ │ = mort  │
     └────┬────┘ └────┬────┘ └────┬────┘
          │           │           │
          └─────┬─────┘           │
                │ "Pret !"        │
          ┌─────▼─────┐           │
          │ Sous-Chef  │◄──────────┘
          │ (3 lenses) │
          │            │
          │ scope      │
          │ securite   │
          │ qualite    │
          │            │
          │ Repair ADN:│
          │ small=skip │
          │ medium=1   │
          │ large=full │
          └────────────┘
```

---

## Comparaison

| Aspect | Brigade complete | Stigmergie |
|--------|-----------------|------------|
| Agents runtime | 6+ (Chef+3SC+Merge+N) | 2+ (1SC+N) |
| Coordination | Messages inter-agents | Lecture/ecriture fichier |
| Point de defaillance | Chef = SPOF | Pas de SPOF |
| Cout tokens | Eleve (Chef pense + 3 votes) | Bas (1 review + N commis) |
| Latence par merge | Haute (vote → resolution → merge) | Basse (review → merge) |
| Audit trail | Explicite (votes, decisions) | Implicite (shared-state.md history, git log) |
| Conformite reglementaire | Oui (tracabilite des votes) | Non (pas de vote auditable) |
| Scalabilite | Limitee par le Chef | Illimitee (ajout de commis) |
