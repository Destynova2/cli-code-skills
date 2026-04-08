# Template — Shared State

Generer ce fichier dans `{project}/.claude/shared-state.md`.
Remplacer les `{variables}` par les valeurs du projet.

---

```markdown
# {Project Name} Shared State — Session {session_name}

> Memoire partagee entre toutes les sessions Claude Code paralleles.
> **BOSS** : seul a ecrire dans "Merges valides" et "Feu vert".
> **WORKERS** : lisent ce fichier AVANT de coder. Ecrivent dans "En cours" et "Fait".
> **TOUS** : relisent ce fichier apres chaque merge pour voir ce qui a change.

## Feu vert — Dependances resolues

> Le boss ecrit ici quand un merge est valide et que les dependants peuvent continuer.
> Workers : si ta tache depend d'une autre, attends qu'elle apparaisse ici.

| Branche mergee | Tests | CI | Date | Debloque |
|----------------|-------|----|------|----------|

## En cours

| Worktree | Branche | Tache | Fichiers touches | Debut |
|----------|---------|-------|------------------|-------|

## Fait (en attente de merge)

| Worktree | Branche | Tache | Resultat | Tests | Fichiers modifies | Date |
|----------|---------|-------|----------|-------|-------------------|------|

## Merges valides

| Branche | Merge commit | CI run | Statut | Date |
|---------|-------------|--------|--------|------|

## Conflits potentiels

> Si tu modifies un fichier present dans "En cours" d'un autre worktree, note-le ici.

| Fichier | Worktrees concernes | Risque | Resolution |
|---------|---------------------|--------|------------|

## Decisions prises

> Choix techniques faits pendant le sprint qui impactent les autres sessions.
> Workers : lisez cette section AVANT de coder, un choix fait par un autre worker peut changer votre approche.

| Decision | Raison | Impact sur | Par | Date |
|----------|--------|-----------|-----|------|

## Quality Gates — Metriques (rempli par le boss)

> Le boss remplit cette section apres chaque audit. Workers : si un gate echoue, corrigez avant de re-soumettre.

| Gate | Scope | Score | Seuil | Statut | Date |
|------|-------|-------|-------|--------|------|
{quality_gates_rows}

### Couplages forts (de /cli-audit-tangle)

> Boss remplit avant d'assigner les taches. Workers : ne touchez PAS aux fichiers couples assignes a un autre worker.

| Module/Fonction | Couplage | Assigne a |
|-----------------|----------|-----------|

## Zones sensibles (quorum 3/3 unanimite)

> Revue a chaque fin de sprint. Si un fichier cause des problemes, le passer ici.
> Si un fichier est ici depuis 3 sprints sans incident, le repasser en zone normale.
>
> **Format machine-parseable** : la liste authoritative est le bloc `BOSS_SENSITIVE_PATHS`
> ci-dessous. Le tableau humain est genere depuis ce bloc et sert a la documentation.
> Le `/loop` Sous-Chef parse uniquement le bloc — toujours le tenir a jour.

<!-- BOSS_SENSITIVE_PATHS:START -->
```sensitive-paths
# One glob pattern per line. Lines starting with # are comments.
# Parsed by /loop Sous-Chef and the Chef shutdown protocol.
# Format: <glob>    # <reason>
.github/workflows/**          # CI = impact global
**/Cargo.toml                 # Supply chain risk (Rust deps)
**/package.json               # Supply chain risk (npm deps)
**/go.mod                     # Supply chain risk (Go deps)
**/pyproject.toml             # Supply chain risk (Python deps)
**/requirements*.txt          # Supply chain risk (Python deps)
.env                          # Secrets
**/*.secret                   # Secrets
**/credentials*               # Secrets
src/auth/**                   # Authentification critique
src/security/**               # Securite critique
CONTRACTS.md                  # Regles du projet
CONTRIBUTING.md               # Instructions du projet
```
<!-- BOSS_SENSITIVE_PATHS:END -->

### Tableau humain (genere depuis le bloc ci-dessus)

| Pattern | Raison | Depuis | Incidents |
|---------|--------|--------|-----------|
| .github/workflows/** | CI = impact global | initial | - |
| **/Cargo.toml | Supply chain risk (Rust) | initial | - |
| **/package.json | Supply chain risk (npm) | initial | - |
| .env, **/*.secret, **/credentials* | Secrets | initial | - |
| src/auth/** | Module authentification critique | initial | - |
| src/security/** | Module securite critique | initial | - |
| CONTRACTS.md, CONTRIBUTING.md | Regles du projet | initial | - |

### Comment parser le bloc (pour /loop ou scripts)

```bash
# Extraire les patterns purs (sans commentaires, sans markers)
sed -n '/<!-- BOSS_SENSITIVE_PATHS:START -->/,/<!-- BOSS_SENSITIVE_PATHS:END -->/p' \
  {project}/.claude/shared-state.md \
  | sed -n '/```sensitive-paths/,/```/p' \
  | grep -v '^```' \
  | grep -v '^#' \
  | grep -v '^$' \
  | awk '{print $1}'
```

### Historique des hallucinations

> Si un Sous-Chef a APPROVE un diff qui a cause un probleme, le noter ici.
> Sert a ameliorer les regles des Sous-Chefs au prochain sprint.

| Sprint | Sous-Chef | Fichier | Ce qui s'est passe | Fix applique |
|--------|-----------|---------|---------------------|--------------|

## Contexte partage

> Infos que tous les workers doivent connaitre.

{context_items}
```
