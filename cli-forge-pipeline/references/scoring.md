# Pipeline Scoring — 15 Dimensions

> **When to read:** After completing the 9-step audit workflow, to score the pipeline.

---

## Scoring Grid

Each dimension is scored 0-4. Total max = 60.

| # | Dimension | 0 | 1 | 2 | 3 | 4 |
|---|-----------|---|---|---|---|---|
| D1 | **DAG** | Tout séquentiel | Quelques `needs:` | DAG partiel | DAG complet | DAG + pruning dynamique |
| D2 | **Cache** | Aucun cache | Cache branche | Cache hash lockfile | Cache waterfall | Cache cross-pipeline + GC |
| D3 | **Parallélisme** | 1 runner | 2-3 jobs // | Matrix/sharding | Fan-out adaptatif | Auto-scale + spot instances |
| D4 | **Résilience** | Pas de retry | Retry aveugle | Retry sélectif (infra) | Fallback registry | Multi-provider + self-healing |
| D5 | **Feedback** | > 15 min | 10-15 min | 5-10 min | 2-5 min | < 2 min (smoke) |
| D6 | **Pruning** | Rebuild tout | paths filter | changes + affected | Merge queue | Predictive skip (ML) |
| D7 | **Artifacts** | Tout partagé | Scoped basique | `needs:` sélectif | Size-optimized | Content-addressed (CAS) |
| D8 | **Security** | Aucun scan | 1 scanner | SAST + deps | + secrets + DAST | + SBOM + signing |
| D9 | **Observabilité** | Logs bruts | Durées par job | Métriques custom | Dashboard temps réel | Anomaly detection |
| D10 | **Mitose** | 1 mega-pipeline | 2 workflows | N workflows scopés | Templates partagés | Event-driven mesh |
| D11 | **Coût** | Pas de mesure | Estimation | Budget alerts | Per-team billing | FinOps optimized |
| D12 | **DX** | Config manuelle | Docs | CLI helpers | Self-service portal | GitOps + preview envs |
| D13 | **Fuzzing** | Aucun | Fuzz ponctuel | Fuzz CI sur parsers | + property-based | Corpus persistant + regression |
| D14 | **Matrix** | 1 env | 2-3 combos | OS × version | + arch × features | Full combinatorial + nightly |
| D15 | **Chaos** | Aucun | Retry = seul filet | Toxiproxy en CI | + disk/mem/time | Chaos Monkey prod + observabilité |

**Score cible :** > 45/60 (75%) pour un projet production.

---

## Scorecard template (copier-coller)

```
Pipeline: _______________
Date: _______________

D1  DAG:           [ ] 0  [ ] 1  [ ] 2  [ ] 3  [ ] 4
D2  Cache:         [ ] 0  [ ] 1  [ ] 2  [ ] 3  [ ] 4
D3  Parallélisme:  [ ] 0  [ ] 1  [ ] 2  [ ] 3  [ ] 4
D4  Résilience:    [ ] 0  [ ] 1  [ ] 2  [ ] 3  [ ] 4
D5  Feedback:      [ ] 0  [ ] 1  [ ] 2  [ ] 3  [ ] 4
D6  Pruning:       [ ] 0  [ ] 1  [ ] 2  [ ] 3  [ ] 4
D7  Artifacts:     [ ] 0  [ ] 1  [ ] 2  [ ] 3  [ ] 4
D8  Security:      [ ] 0  [ ] 1  [ ] 2  [ ] 3  [ ] 4
D9  Observabilité: [ ] 0  [ ] 1  [ ] 2  [ ] 3  [ ] 4
D10 Mitose:        [ ] 0  [ ] 1  [ ] 2  [ ] 3  [ ] 4
D11 Coût:          [ ] 0  [ ] 1  [ ] 2  [ ] 3  [ ] 4
D12 DX:            [ ] 0  [ ] 1  [ ] 2  [ ] 3  [ ] 4
D13 Fuzzing:       [ ] 0  [ ] 1  [ ] 2  [ ] 3  [ ] 4
D14 Matrix:        [ ] 0  [ ] 1  [ ] 2  [ ] 3  [ ] 4
D15 Chaos:         [ ] 0  [ ] 1  [ ] 2  [ ] 3  [ ] 4

TOTAL: ___/60  (___%)
```

---

## Benchmarks

| Type de projet | Score typique | Score cible |
|---------------|--------------|-------------|
| Startup MVP | 10-20 | 25+ |
| SaaS standard | 20-35 | 40+ |
| Enterprise | 30-45 | 50+ |
| Safety-critical | 40-50 | 55+ |
