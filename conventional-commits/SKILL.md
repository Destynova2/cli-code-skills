---
name: conventional-commits
description: >
  ALWAYS apply this skill for any git or jj operation involving commits, branches, tags, or releases.
  Trigger on: "commit", "message de commit", "crée une branche", "propose un commit", "quel tag",
  "quelle version", "release", "changelog", "semver", "conventional commit", diff partagé sans message,
  "git tag", "jj describe", "jj branch", "bump version", "next version", "CHANGELOG.md".
  Also trigger proactively when the user shares code changes and hasn't mentioned what to commit.
  This skill enforces Conventional Commits v1.0.0 to the letter, SemVer 2.0.0, branch naming
  conventions, git/jj commands, and writes commits in a HUMAN GHOSTWRITER style — natural, direct,
  zero AI markers. NEVER adds Claude as co-author: the sole author is always the person who asked.
---

# Conventional Commits — Skill

Référence canonique : https://www.conventionalcommits.org/en/v1.0.0/

---

## Règle absolue : Authorship

**JAMAIS de `Co-authored-by: Claude` ni aucun trailer qui mentionne Claude, Anthropic ou un outil IA.**

Le commit appartient entièrement à la personne qui a demandé. Elle en est la seule responsable.
Ne pas ajouter de trailers `Co-authored-by`, `Generated-by`, `AI-assisted` ou équivalents.

---

## Style ghostwriter : écrire comme un humain

Claude est le **ghostwriter** du commit. Le message est signé par toi, écrit dans ta voix.

### Règles de style

| ✗ Style AI à éviter | ✓ Style humain |
|---------------------|----------------|
| `feat: implement comprehensive solution for handling edge cases in authentication module` | `feat(auth): handle token expiry on refresh` |
| `refactor: enhance code quality and maintainability by restructuring` | `refactor(router): split dispatch logic into smaller fns` |
| `fix: resolve critical issue that was causing unexpected behavior` | `fix(cache): prevent stale reads after TTL reset` |
| `chore: update various dependencies to their latest versions` | `chore(deps): bump tokio 1.35 → 1.37` |
| Phrases avec "comprehensive", "robust", "leverage", "utilize" | Verbes directs : add, fix, remove, split, bump, wire, drop |
| Description qui reformule le type (`fix: fix the bug`) | Description qui précise le quoi/pourquoi |

### Ce qui rend un commit authentiquement humain

1. **Précis, pas exhaustif** — un commit dit une chose, pas tout ce qu'il fait
2. **Impératif présent** — `add`, `fix`, `remove` (jamais `added`, `fixing`, `removes`)
3. **Concret** — nomme les fichiers, fonctions, routes concernés si pertinent
4. **Corps = raisonnement** — pourquoi ce choix, pas la description du diff
5. **Zéro remplissage** — pas de "in order to", "as part of", "to ensure that"
6. **Vocabulary du projet** — utilise les termes du codebase, pas des synonymes génériques

### Exemples de corps bien écrits

```
fix(dlp): reject requests exceeding 64KB payload limit

Previous limit was soft and enforced only at the handler layer.
Some fan-out paths bypassed it. Now enforced at middleware entry.
```

```
refactor(routing): extract classifier into dedicated module

Was getting unwieldy in main.rs. Classifier logic is now in
src/classifier.rs with its own test suite.
```

```
feat(audit): add ECDSA-P256 signature on log entries

Signed logs allow downstream verifiers to detect tampering
without trusting the logging pipeline itself.
```

---

## SemVer 2.0.0 — Semantic Versioning

Référence : https://semver.org/

### Format

```
MAJOR.MINOR.PATCH[-pre-release][+build-metadata]
```

### Correspondance Conventional Commits → SemVer

| Commit | Bump SemVer |
|--------|-------------|
| `fix:` | PATCH `x.y.Z+1` |
| `feat:` | MINOR `x.Y+1.0` |
| `BREAKING CHANGE` ou `!` | MAJOR `X+1.0.0` |
| `docs:`, `chore:`, `ci:`, `style:`, `refactor:`, `test:` | aucun bump |
| `perf:` | PATCH (amélioration sans cassure) |

### Règles MUST (SemVer spec)

- `0.y.z` = développement initial, API instable, tout peut changer
- `1.0.0` = première API publique stable déclarée
- Une version publiée ne peut **jamais** être modifiée — publier une nouvelle version
- Le PATCH est remis à 0 quand MINOR est incrémenté
- MINOR et PATCH sont remis à 0 quand MAJOR est incrémenté
- Les pre-releases ont précédence inférieure : `1.0.0-alpha < 1.0.0-beta < 1.0.0-rc.1 < 1.0.0`
- Les build metadata (`+001`, `+sha.5114f85`) n'affectent pas la précédence

### Pre-release identifiers

```
1.0.0-alpha        # premier test interne
1.0.0-alpha.1      # itération alpha
1.0.0-beta         # fonctionnellement complet, instable
1.0.0-rc.1         # release candidate
1.0.0              # release stable
```

### Git tags SemVer

```bash
# Toujours annoté (pas lightweight) — conserve l'auteur, la date, le message
git tag -a v1.2.3 -m "release: v1.2.3"
git push origin v1.2.3

# Lister les tags ordonnés
git tag --sort=-version:refname | head -10

# Dernier tag SemVer
git describe --tags --abbrev=0

# Tag d'une release candidate
git tag -a v2.0.0-rc.1 -m "release: v2.0.0-rc.1 — candidate for major API rewrite"
```

### Résoudre le prochain bump

Quand on te demande "quelle version ensuite ?" :

1. Récupère le dernier tag : `git describe --tags --abbrev=0`
2. Parcours les commits depuis ce tag : `git log <last-tag>..HEAD --oneline`
3. Applique la règle la plus haute trouvée :
   - Si un commit a `BREAKING CHANGE` ou `!` → bump MAJOR
   - Sinon si au moins un `feat:` → bump MINOR
   - Sinon si au moins un `fix:` ou `perf:` → bump PATCH
   - Sinon → pas de release, ou release patch de maintenance

### CHANGELOG.md

Structure recommandée (format [Keep a Changelog](https://keepachangelog.com)) :

```markdown
# Changelog

## [Unreleased]

## [1.2.0] - 2026-04-05
### Added
- feat(auth): PKCE support for OAuth2 flow

### Fixed
- fix(cache): prevent stale reads after TTL reset

## [1.1.0] - 2026-02-14
...
```

---

## Format du message de commit

```
<type>[portée optionnelle][! optionnel]: <description>

[corps optionnel]

[footer(s) optionnel(s)]
```

### Règles MUST (spec RFC 2119)

1. Le message DOIT commencer par un `type` (nom), suivi d'un `:` et d'un espace.
2. `feat` DOIT être utilisé quand le commit ajoute une nouvelle fonctionnalité.
3. `fix` DOIT être utilisé quand le commit corrige un bug.
4. La portée (scope) est OPTIONNELLE, entre parenthèses après le type : `fix(auth):`.
5. Une description courte DOIT suivre immédiatement le `: ` — jamais vide.
6. Le corps est OPTIONNEL, séparé de la description par **une ligne vide**.
7. Les footers sont OPTIONNELS, séparés du corps (ou de la description) par **une ligne vide**.
8. Chaque footer DOIT avoir la forme `Token: valeur` ou `Token #valeur`.
9. Les tokens multi-mots utilisent `-` comme séparateur : `Reviewed-by:`, `Refs:`.
10. `BREAKING CHANGE` est l'unique exception : le token peut avoir un espace.
11. Un breaking change DOIT être signalé soit par `!` avant `:`, soit par le footer `BREAKING CHANGE: description` (ou les deux).
12. `BREAKING-CHANGE` est synonyme de `BREAKING CHANGE` en footer.
13. La casse des types est libre mais DOIT être cohérente dans le projet. Recommandé : **lowercase**.

---

## Types autorisés

| Type       | SemVer impact | Usage |
|------------|---------------|-------|
| `feat`     | MINOR         | Nouvelle fonctionnalité |
| `fix`      | PATCH         | Correction de bug |
| `build`    | —             | Système de build, dépendances externes |
| `chore`    | —             | Tâches de maintenance sans impact fonctionnel |
| `ci`       | —             | Configuration CI/CD |
| `docs`     | —             | Documentation uniquement |
| `style`    | —             | Formatage, whitespace (pas de changement logique) |
| `refactor` | —             | Restructuration sans feat ni fix |
| `perf`     | —             | Optimisation de performance |
| `test`     | —             | Ajout ou modification de tests |
| `revert`   | —             | Annulation d'un commit précédent |

> **Règle** : si le changement couvre plusieurs types, **découper en plusieurs commits**.

---

## Description (subject line)

- Impératif présent, anglais : `add`, `fix`, `remove`, pas `added`, `fixes`, `removing`
- Pas de majuscule initiale : `feat: add retry logic` ✓ — `feat: Add retry logic` ✗
- Pas de point final
- Longueur max : **72 caractères** (subject line entière, type compris)
- Précis et factuel : **quoi** + éventuellement **pourquoi** en une phrase

---

## Corps (body)

- Séparé de la description par **une ligne vide obligatoire**
- Expliquer le **pourquoi**, pas le comment (le code montre le comment)
- Paragraphes libres, retours à la ligne à 72 caractères
- Peut référencer des issues, tickets, PRs

---

## Footers

```
BREAKING CHANGE: <description de ce qui casse>
Refs: #123, #456
Reviewed-by: Alice <alice@example.com>
Closes: #789
```

- Chaque footer sur sa propre ligne
- **Aucun** `Co-authored-by: Claude` ou similaire

---

## Exemples canoniques

### Feat simple
```
feat(auth): add PKCE support for OAuth2 flow
```

### Fix avec refs
```
fix(router): prevent nil dereference on empty path

Refs: #42
```

### Breaking change avec `!`
```
feat(api)!: remove legacy v1 endpoints

BREAKING CHANGE: /api/v1/* routes are no longer served. Migrate to /api/v2/*.
```

### Revert
```
revert: feat(cache): add in-memory LRU cache

Refs: a1b2c3d, e4f5g6h
```

### Chore deps
```
chore(deps): bump tokio from 1.35 to 1.37
```

---

## Nommage de branches

### Pattern recommandé

```
<type>/<description-kebab-case>
```

### Règles

- Lowercase uniquement
- Kebab-case (`-`), pas de `_` ni d'espaces
- `/` comme séparateur type/description
- Description courte et explicite (3–5 mots max)
- Pas de numéro de ticket seul — préférer une description lisible
- Optionnel : préfixer avec l'ID de ticket : `feat/GH-123-add-dlp-middleware`

### Exemples

```
feat/add-dlp-middleware
fix/auth-token-expiry
chore/update-rust-deps
ci/add-arm64-runner
docs/update-api-reference
refactor/split-router-module
feat/GH-42-fan-out-racing
```

### Types de branches spéciaux

| Branche     | Usage |
|-------------|-------|
| `main`      | Branche principale, production-ready |
| `develop`   | Intégration (si workflow gitflow) |
| `release/x.y.z` | Préparation d'une release |
| `hotfix/<description>` | Fix critique sur main |

---

## Git — Commandes

### Commit atomique propre
```bash
git add -p                          # staging interactif, commit atomique
git commit -m "feat(scope): description"

# Avec body + footer
git commit -m "fix(auth): handle expired tokens

Remove the silent swallow of TokenExpiredError. Callers now
receive a 401 with Retry-After header.

Refs: #88"
```

### Modifier le dernier commit (avant push)
```bash
git commit --amend --no-edit        # ajouter des fichiers oubliés
git commit --amend -m "nouveau message"
```

### Rebase interactif pour nettoyer l'historique
```bash
git rebase -i HEAD~3                # réécrire les 3 derniers commits
# commandes utiles dans l'éditeur : reword, squash, fixup, drop
```

### Branch
```bash
git switch -c feat/add-dlp-middleware
git push -u origin feat/add-dlp-middleware
```

### Vérifier avant de commit
```bash
git diff --staged                   # inspecter ce qui va partir
git log --oneline -10               # contexte de l'historique
```

---

## jj (Jujutsu) — Commandes

### Décrire le commit courant
```bash
jj describe -m "feat(scope): description"

# Avec body multi-ligne
jj describe -m "fix(auth): handle expired tokens

Remove the silent swallow of TokenExpiredError. Callers now
receive a 401 with Retry-After header.

Refs: #88"
```

### Nouveau commit vide (workflow jj natif)
```bash
jj new -m "feat(scope): description"
```

### Branches (bookmarks)
```bash
jj bookmark create feat/add-dlp-middleware
jj bookmark set feat/add-dlp-middleware          # déplacer au commit courant
jj git push --bookmark feat/add-dlp-middleware
```

### Modifier un commit passé (sans rebase interactif)
```bash
jj edit <rev>                       # se placer sur le commit à modifier
jj describe -m "nouveau message"    # réécrire le message
jj new @+                           # revenir au descendant
```

### Squash
```bash
jj squash                           # fusionner le commit courant dans son parent
jj squash --into <rev>              # fusionner dans un commit spécifique
```

---

## Anti-patterns à rejeter

| ✗ À ne pas faire | ✓ Correct |
|------------------|-----------|
| `fix stuff` | `fix(cache): prevent stale entries after TTL expiry` |
| `WIP` | Découper en commits atomiques avec messages propres |
| `feat: Added new feature` | `feat: add retry backoff on transient errors` |
| `Update` / `Changes` | Message explicite avec type |
| Tout dans `chore` | Utiliser le type approprié |
| Scope trop large : `feat(app):` | Scope précis : `feat(router):` |
| Commit avec 12 fichiers non liés | Commits atomiques, un changement logique par commit |

---

## Checklist avant de livrer un message de commit

- [ ] Type correct parmi la liste autorisée
- [ ] Scope pertinent (ou absent si non applicable)
- [ ] `!` et/ou footer `BREAKING CHANGE` si cassant
- [ ] Description : impératif présent, lowercase, pas de point, ≤72 chars
- [ ] Corps séparé par ligne vide si présent
- [ ] Footers valides (token: valeur)
- [ ] **Aucun** trailer Claude / IA / co-author automatique
- [ ] Commit atomique (un seul changement logique)
