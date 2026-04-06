# Gotchas — Multi-Agent Boss Orchestration

> **Rule:** The boss prompt MUST include this section verbatim. Every gotcha comes from a real failure.

---

## G1 — Permission UI bloque le boss (CRITIQUE)

**Probleme:** Meme avec `--dangerously-skip-permissions`, le boss (team leader) recoit des prompts UI interactifs pour approuver les edits des workers. Le boss se bloque sur "Do you want to make this edit?" et attend un Enter humain.

**Cause:** Agent Teams design — le team leader est le gatekeeper des permissions des teammates. Le flag `--dangerously-skip-permissions` bypass les permissions du boss lui-meme, mais PAS celles des workers qui remontent au leader.

**Fix:** Ajouter `--permission-mode bypassPermissions` au boss en PLUS de `--dangerously-skip-permissions` :
```yaml
claude --dangerously-skip-permissions --permission-mode bypassPermissions --teammate-mode tmux
```

**Si oublie :** Le boss va se bloquer sur chaque edit de worker. Il faudra approuver manuellement avec Enter dans le pane boss.

---

## G2 — --system-prompt-file n'existe pas

**Probleme:** Le flag `--system-prompt-file` n'existe pas dans Claude Code CLI. Le boss ne se lance pas.

**Fix:** Utiliser `--append-system-prompt "$(cat path/to/prompt.md)"` pour charger un fichier prompt.

**Alternative:** `--system-prompt "contenu inline"` mais limite pour les gros prompts.

---

## G3 — Claude attend un premier message

**Probleme:** Claude Code en mode interactif attend toujours un premier message utilisateur. Le boss est lance mais ne fait rien.

**Fix:** Inclure dans le boss prompt une instruction claire de demarrage autonome. Si ca ne suffit pas, l'utilisateur tape dans le pane boss ou on envoie via tmux :
```bash
tmux send-keys -t {session}:boss "Lance le plan complet." Enter
```

**NE PAS utiliser `-p` (print mode)** — ca desactive le mode interactif et les panes tmux des teammates ne s'afficheront pas.

---

## G4 — Workers invisibles sans --teammate-mode tmux

**Probleme:** Par defaut, les teammates Agent Teams sont des subprocesses invisibles. L'utilisateur ne voit rien dans tmux, juste le boss.

**Fix:** Toujours ajouter `--teammate-mode tmux` au boss :
```yaml
claude --teammate-mode tmux
```

**Alternative permanente:** Mettre `"teammateMode": "tmux"` dans `~/.claude.json`.

---

## G5 — shared-state.md hors du worktree = permission

**Probleme:** Les workers dans des worktrees (`grob-wt-hit/`) veulent editer `shared-state.md` qui est dans le repo principal (`grob/.claude/`). Ca declenche une permission request vers le boss.

**Fix:** Ajouter dans `settings.local.json` :
```json
"Edit(//path/to/project/.claude/shared-state.md)"
```

Et donner le chemin absolu dans les prompts workers :
```
MEMOIRE PARTAGEE : Lis /chemin/absolu/projet/.claude/shared-state.md
```

---

## G6 — Auto-approve Enter trop rapide

**Probleme:** Envoyer des `Enter` via tmux send-keys trop vite (< 1s) — Claude UI les ignore car il n'a pas eu le temps de render le prompt.

**Fix:** Si on doit auto-approve, espacer de 3s minimum :
```bash
for i in $(seq 1 N); do tmux send-keys -t session:boss Enter; sleep 3; done
```

**Mieux:** Utiliser G1 (`--permission-mode bypassPermissions`) pour ne jamais avoir de prompts.

---

## G7 — tmuxinator ne peut pas lancer Claude en background

**Probleme:** Tenter de lancer `claude &` dans tmuxinator ou de faire un `bash -c 'claude & sleep...'` casse le TTY. Claude a besoin d'un vrai terminal.

**Fix:** Laisser tmuxinator lancer Claude normalement (foreground dans le pane). Ne pas essayer de backgrounder.

---

## G8 — Workers recoivent le mauvais prompt

**Probleme:** Les auto-approve `Enter` envoyes au boss peuvent atterrir dans le prompt input d'un worker si le focus tmux a change, ou le message de lancement du boss se retrouve dans un pane worker.

**Fix:** Toujours cibler le bon pane :
```bash
tmux send-keys -t {session}:0.0 Enter  # boss = toujours pane 0.0
```

---

## G9 — Merge sans CI = catastrophe

**Probleme:** Le boss merge dans develop sans attendre la CI. La branche develop est cassee.

**Fix:** Le boss prompt DOIT contenir :
```
CI verte OBLIGATOIRE entre chaque merge.
Boucler sur: gh run watch
```

---

## G10 — Workers touchent les memes fichiers

**Probleme:** Deux workers modifient le meme fichier (ex: `mod.rs`, `Cargo.toml`) → conflit de merge garanti.

**Fix:**
1. Phase 0 : `/cli-audit-tangle` pour identifier les couplages
2. Assigner les fichiers couples au MEME worker
3. Section "Conflits potentiels" dans shared-state.md
4. Workers lisent "En cours" AVANT de coder

---

## G11 — Trop de workers = context window

**Probleme:** > 5 workers en parallele. Le boss depense trop de tokens a gerer les messages/permissions, et les workers se marchent dessus.

**Fix:** Maximum 5 workers. Si plus de 5 taches, sequencer en phases.

---

## G12 — on_project_first_start n'existe pas dans tmuxinator

**Probleme:** `on_project_first_start` n'est pas un hook valide tmuxinator. Le YAML est silencieusement ignore.

**Fix:** Utiliser `on_project_start` (se lance a chaque start). Rendre les commandes idempotentes avec `2>/dev/null || true`.

---

## G13 — Worktree sur branche existante

**Probleme:** `git worktree add ../wt -b feat/x` echoue si la branche existe deja.

**Fix:** Rendre idempotent :
```bash
git worktree add ../wt -b feat/x 2>/dev/null || true
```

---

## G18 — brew upgrade tue les workers en vol

**Probleme:** Agent Teams resout le symlink `claude` en chemin absolu au moment du spawn (ex: `Caskroom/claude-code/2.1.84/claude`). Si `brew upgrade claude-code` tourne pendant la session, le chemin pointe vers une version supprimee. Les nouveaux workers crashent avec "Aucun fichier ou dossier".

**Fix:** Ne JAMAIS faire `brew upgrade claude-code` pendant une session multi-agent. Si un upgrade a eu lieu : relancer le boss (`tmuxinator stop && tmuxinator start`).

**Detection:** Si un worker meurt avec `env: ... Aucun fichier ou dossier`, verifier `ls /home/linuxbrew/.linuxbrew/Caskroom/claude-code/` — si la version du spawn n'y est plus, c'est ca.

---

## G17 — Le Chef oublie de mettre a jour la roadmap

**Probleme:** Le Chef code, merge, fait le rapport, mais ne met pas a jour la documentation du projet (roadmap, features, design docs). Le sprint suivant commence avec des docs obsoletes.

**Fix:** Dans le shutdown protocol du Chef, etape OBLIGATOIRE avant le rapport final :
1. Mettre a jour la roadmap (marquer FAIT, ajouter prochaines etapes)
2. Mettre a jour le features doc (nouvelles fonctionnalites)
3. Mettre a jour les design docs (statuts PROPOSED → DONE)
4. Si Obsidian : mettre a jour les fichiers .md du vault

**Le rapport final ne peut pas etre produit avant que les docs soient a jour.**

---

## G16 — Auto-approve aveugle = zero gardefou

**Probleme:** Pour debloquer le boss qui attend les permissions des workers, on spam Enter en boucle. Ca approuve tout sans verification — un worker pourrait editer un fichier sensible, pousser du code malveillant, ou toucher des fichiers hors de son scope.

**Fix:** Ne JAMAIS auto-approve aveuglement. Le Sous-Chef doit lire le diff avant d'approuver. Regles du Sous-Chef :
- **APPROVE** : edit dans le scope du worker (ses fichiers assignes + shared-state.md)
- **DENY** : edit hors scope (fichier d'un autre worker, .env, credentials, ci.yml sans demande)
- **DENY** : suppression de tests ou de security checks
- **DENY** : modification de Cargo.toml deps sans justification
- **ESCALE** : si doute, demander au Chef

**Si pas de Sous-Chef** (session 2-tiers) : l'utilisateur est le Sous-Chef. Il regarde le pane boss et approuve/refuse manuellement.

---

## G15 — PRs paralleles sur le meme fichier = conflit

**Probleme:** Le chef ouvre N PRs en parallele qui touchent le meme fichier (ex: ci.yml). La premiere merge OK, les suivantes ont des conflits.

**Fix:** Si plusieurs taches touchent le meme fichier, les sequencer dans le PERT (pas en parallele). Ou mieux : merger localement dans l'ordre, tester, puis push une seule PR.

**Regle pour le sous-chef :** Avant de merger une PR, verifier si une autre PR ouverte touche le meme fichier. Si oui, merger la premiere, rebase la seconde, puis merger.

---

## G14 — Read access aux repos externes

**Probleme:** Un worker doit lire un autre repo (ex: sokolsky) mais n'a pas les permissions.

**Fix:** Ajouter dans `settings.local.json` :
```json
"Read(//path/to/other/repo/**)"
```

---

## G19 — Agent Teams non active = pas de TeamCreate/SendMessage

**Probleme:** Le skill genere un prompt avec TeamCreate mais Agent Teams n'est pas active. Le boss n'a pas les outils.

**Fix automatique dans le skill (Phase 0.3)** :
1. Verifier `~/.claude/settings.json` pour `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS`
2. Verifier `~/.claude.json` pour `teammateMode`
3. Si absent : les ajouter automatiquement AVANT de generer le tmuxinator
4. Verifier `skipDangerousModePermissionPrompt` dans `~/.claude/settings.json`

```bash
# Check + fix automatique
python3 -c "
import json, os
p = os.path.expanduser('~/.claude/settings.json')
d = json.load(open(p)) if os.path.exists(p) else {}
d.setdefault('env', {})['CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS'] = '1'
d['skipDangerousModePermissionPrompt'] = True
json.dump(d, open(p, 'w'), indent=2)
"
```

---

## G20 — Le Chef doit envoyer le premier message manuellement

**Probleme:** Claude Code en interactif attend toujours un message. Le tmuxinator lance Claude, mais rien ne se passe. L'utilisateur doit aller dans le pane et taper "Lance le plan".

**Fix dans le tmuxinator** : ajouter un `on_project_start` qui envoie le premier message apres un delai :

```yaml
on_project_start:
  # ... worktree setup ...
  # Auto-kick le chef apres 12s (temps de boot Claude)
  - sleep 12 && tmux send-keys -t {session}:conductor "Lance le service complet. Pilotage autonome." Enter &
```

Le `&` en fin de commande lance en background — tmuxinator ne bloque pas dessus.

---

## G21 — Le boss idle pense que les workers tournent

**Probleme:** Les workers sont morts (G18, crash, timeout) mais le boss affiche `Idle · teammates running`. Il attend des rapports qui ne viendront jamais.

**Fix dans le prompt du Chef** : ajouter un watchdog :

```
WATCHDOG : Si tu es idle depuis plus de 10 minutes et qu'aucun worker ne t'a envoye de message, verifie s'ils sont encore vivants. Essaie de leur envoyer un ping :
  SendMessage { to: "worker-X", summary: "ping", message: "Es-tu vivant? Reponds." }
Si pas de reponse en 30s : le worker est mort. Fais le travail toi-meme ou relance.
```

---

## G22 — Obsidian docs pas dans le scope des permissions

**Probleme:** Le Chef doit mettre a jour les docs Obsidian (roadmap, features) mais le vault Obsidian est hors du projet. Les edits sont bloques par les permissions.

**Fix dans settings.local.json** : ajouter les chemins Obsidian :

```json
"Edit(//{obsidian_vault_path}/**)",
"Read(//{obsidian_vault_path}/**)"
```

---

## G23 — Pas de fallback quand les workers crashent

**Probleme:** Si tous les workers sont morts (G18) et qu'on ne peut pas relancer tmuxinator, le sprint est bloque.

**Fix dans le prompt du Chef** : plan B explicite :

```
SI LES WORKERS SONT MORTS :
1. Ne PAS essayer de re-spawner (meme erreur)
2. Faire le travail TOI-MEME directement dans cette session
3. Utiliser les worktrees existants (cd /path/to/worktree)
4. Un seul job a la fois (pas de parallelisme) mais ca avance
5. Signaler a l'utilisateur : "Workers morts, je continue en solo"
```
