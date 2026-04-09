# Pipeline Scoring — 15 Dimensions

> **When to read:** After completing the 9-step audit workflow, to score the pipeline.

---

## Scoring Grid

Each dimension is scored 0-4. Total max = 60.

| # | Dimension | 0 | 1 | 2 | 3 | 4 |
|---|-----------|---|---|---|---|---|
| D1 | **DAG** | All sequential | A few `needs:` | Partial DAG | Full DAG | DAG + dynamic pruning |
| D2 | **Cache** | No cache | Branch cache | Hashed-lockfile cache | Waterfall cache | Cross-pipeline cache + GC |
| D3 | **Parallelism** | 1 runner | 2-3 parallel jobs | Matrix/sharding | Adaptive fan-out | Auto-scale + spot instances |
| D4 | **Resilience** | No retry | Blind retry | Selective retry (infra) | Registry fallback | Multi-provider + self-healing |
| D5 | **Feedback** | > 15 min | 10-15 min | 5-10 min | 2-5 min | < 2 min (smoke) |
| D6 | **Pruning** | Rebuild everything | paths filter | changes + affected | Merge queue | Predictive skip (ML) |
| D7 | **Artifacts** | Everything shared | Basic scoping | Selective `needs:` | Size-optimized | Content-addressed (CAS) |
| D8 | **Security** | No scan | 1 scanner | SAST + deps | + secrets + DAST | + SBOM + signing |
| D9 | **Observability** | Raw logs | Per-job durations | Custom metrics | Real-time dashboard | Anomaly detection |
| D10 | **Mitosis** | 1 mega-pipeline | 2 workflows | N scoped workflows | Shared templates | Event-driven mesh |
| D11 | **Cost** | No measurement | Estimation | Budget alerts | Per-team billing | FinOps optimized |
| D12 | **DX** | Manual config | Docs | CLI helpers | Self-service portal | GitOps + preview envs |
| D13 | **Fuzzing** | None | Occasional fuzz | Fuzz in CI on parsers | + property-based | Persistent corpus + regression |
| D14 | **Matrix** | 1 env | 2-3 combos | OS × version | + arch × features | Full combinatorial + nightly |
| D15 | **Chaos** | None | Retry = only safety net | Toxiproxy in CI | + disk/mem/time | Chaos Monkey in prod + observability |

**Target score:** > 45/60 (75%) for a production project.

---

## Scorecard template (copier-coller)

```
Pipeline: _______________
Date: _______________

D1  DAG:           [ ] 0  [ ] 1  [ ] 2  [ ] 3  [ ] 4
D2  Cache:         [ ] 0  [ ] 1  [ ] 2  [ ] 3  [ ] 4
D3  Parallelism:   [ ] 0  [ ] 1  [ ] 2  [ ] 3  [ ] 4
D4  Resilience:    [ ] 0  [ ] 1  [ ] 2  [ ] 3  [ ] 4
D5  Feedback:      [ ] 0  [ ] 1  [ ] 2  [ ] 3  [ ] 4
D6  Pruning:       [ ] 0  [ ] 1  [ ] 2  [ ] 3  [ ] 4
D7  Artifacts:     [ ] 0  [ ] 1  [ ] 2  [ ] 3  [ ] 4
D8  Security:      [ ] 0  [ ] 1  [ ] 2  [ ] 3  [ ] 4
D9  Observability: [ ] 0  [ ] 1  [ ] 2  [ ] 3  [ ] 4
D10 Mitosis:       [ ] 0  [ ] 1  [ ] 2  [ ] 3  [ ] 4
D11 Cost:          [ ] 0  [ ] 1  [ ] 2  [ ] 3  [ ] 4
D12 DX:            [ ] 0  [ ] 1  [ ] 2  [ ] 3  [ ] 4
D13 Fuzzing:       [ ] 0  [ ] 1  [ ] 2  [ ] 3  [ ] 4
D14 Matrix:        [ ] 0  [ ] 1  [ ] 2  [ ] 3  [ ] 4
D15 Chaos:         [ ] 0  [ ] 1  [ ] 2  [ ] 3  [ ] 4

TOTAL: ___/60  (___%)
```

---

## Benchmarks

| Project type | Typical score | Target score |
|--------------|--------------|--------------|
| Startup MVP | 10-20 | 25+ |
| Standard SaaS | 20-35 | 40+ |
| Enterprise | 30-45 | 50+ |
| Safety-critical | 40-50 | 55+ |
