# Quality Gates — Skills CLI pour merge gates

## Gates disponibles

| Skill | Quand | Bloque si | Cout tokens |
|-------|-------|-----------|-------------|
| `/cli-audit-tangle` | Pre-cycle (Phase 0) | God functions, cycles | Eleve |
| `/cli-audit-drift` | Par merge | Loupers (code ≠ contrat CONTRACTS.md) | Moyen |
| `/cli-audit-code` | Par merge | CQI baisse > 0.2 | Moyen |
| `/cli-audit-test` | Par merge | Pyramide tests desequilibree | Moyen |
| `/cli-audit-sync` | Post-merge (si docs) | Docs desynchronisees | Moyen |
| `/cli-cycle` | Fin de cycle | Regression globale | Eleve |

## Quand utiliser quoi

### Minimum viable (tout projet)

```
Phase 0 : /cli-audit-tangle -> assigner les modules
Par merge : /cli-audit-code -> qualite minimum
Fin cycle : /cli-cycle -> scorecard
```

### Standard (projet avec contrats/docs)

```
Phase 0 : /cli-audit-tangle
Par merge : /cli-audit-code + /cli-audit-drift
Post-merge docs : /cli-audit-sync
Fin cycle : /cli-cycle
```

### Complet (projet enterprise/regulated)

```
Phase 0 : /cli-audit-tangle
Par merge : /cli-audit-code + /cli-audit-drift + /cli-audit-test
Post-merge docs : /cli-audit-sync
Fin cycle : /cli-cycle
```

## Integration dans le boss prompt

```markdown
## Quality Gates

### Protocole

1. Worker envoie "Tache terminee" via SendMessage
2. Boss execute les gates dans gate :
   /cli-audit-code {scope}
   /cli-audit-drift {scope}
3. Si gate echoue :
   SendMessage { to: "{worker}", summary: "gate failed", message: "GATE FAILED: {detail}. Corrige et recommit." }
4. Attendre le fix, re-executer le gate
5. Si gate passe : merger
```

## Protocole si gate echoue

1. Lire le rapport d'audit
2. Identifier les fichiers/fonctions en cause
3. Envoyer au worker concerne un message precis :
   - Quel gate a echoue
   - Quel fichier/fonction est en cause
   - Ce que le contrat/regle attendait
   - Ce que le code fait a la place
4. Attendre le fix du worker
5. Re-executer le gate
6. Si 3 echecs consecutifs : escalader a l'utilisateur
