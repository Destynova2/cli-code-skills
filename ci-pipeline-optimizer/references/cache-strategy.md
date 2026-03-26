# Cache Strategy — Blob + Mycélium Combinés

## Principe : le cache est le système nerveux de la colonie

Sans mémoire partagée, chaque fourmi repart de zéro.
Le cache CI = les phéromones et le mycélium combinés.

---

## Niveaux de cache (waterfall mycélium)

```
Job sur feature/xyz
  └─ Cherche cache key: deps-feature-xyz         → MISS
      └─ Cherche cache key: deps-main             → HIT ✓ (partial restore)
          └─ Sinon image Docker pré-buildée       → MISS
              └─ Sinon reconstruction complète    → SLOW PATH
```

### GitLab CI — Cache waterfall

```yaml
cache:
  - key:
      files: [package-lock.json]
      prefix: node-$CI_COMMIT_REF_SLUG
    paths: [node_modules/]
    policy: pull-push          # Branche courante : lit + écrit

  - key:
      files: [package-lock.json]
      prefix: node-main
    paths: [node_modules/]
    policy: pull               # Fallback main : lit seulement

  - key: node-base-image
    paths: [node_modules/]
    policy: pull               # Fallback ultime : lit seulement
```

---

## Cache key strategies par écosystème

### Rust / Cargo
```yaml
cache:
  key:
    files: [Cargo.lock]
  paths:
    - target/
    - ~/.cargo/registry/
    - ~/.cargo/git/
```

### Node.js
```yaml
cache:
  key:
    files: [package-lock.json]
  paths: [node_modules/]
```

### Python / pip
```yaml
cache:
  key:
    files: [requirements.txt, pyproject.toml]
  paths: [.pip-cache/]
  variables:
    PIP_CACHE_DIR: $CI_PROJECT_DIR/.pip-cache
```

### Go
```yaml
cache:
  key:
    files: [go.sum]
  paths:
    - $GOPATH/pkg/mod/
    - .go-build-cache/
  variables:
    GOMODCACHE: $CI_PROJECT_DIR/.go-mod-cache
    GOCACHE: $CI_PROJECT_DIR/.go-build-cache
```

### Docker layers (Physarum : renforcer les tubes utilisés)
```yaml
build-image:
  script:
    - docker buildx build
        --cache-from type=registry,ref=$CI_REGISTRY_IMAGE:cache
        --cache-to type=registry,ref=$CI_REGISTRY_IMAGE:cache,mode=max
        --tag $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
        .
```

---

## Stratégie Physarum : cache adaptatif par profil de changement

Le blob renforce les tubes qui servent, prune les autres.

```yaml
# Jobs touchés par le frontend seulement
.frontend-cache:
  cache:
    key:
      files: [frontend/package-lock.json]
      prefix: frontend
    paths: [frontend/node_modules/]

# Jobs touchés par le backend seulement
.backend-cache:
  cache:
    key:
      files: [backend/go.sum]
      prefix: backend
    paths: [backend/.go-mod-cache/]

test-frontend:
  extends: .frontend-cache
  rules:
    - changes: [frontend/**/*]

test-backend:
  extends: .backend-cache
  rules:
    - changes: [backend/**/*]
```

---

## Pièges courants

### ❌ Cache key trop large → cache inutile
```yaml
# MAUVAIS : invalide à chaque commit
cache:
  key: "$CI_COMMIT_SHA"   # Jamais de hit

# MAUVAIS : invalide à chaque branche
cache:
  key: "$CI_COMMIT_REF_NAME"  # Branches éphémères = cache éphémère
```

### ❌ Cache non-scoped → pollution entre projets
```yaml
# MAUVAIS sur runner partagé
cache:
  paths: [node_modules/]
  # Clé par défaut = $CI_JOB_NAME, OK mais risque de collision
```

### ✅ Cache scoped et content-based
```yaml
cache:
  key:
    files: [package-lock.json]
    prefix: $CI_PROJECT_PATH_SLUG  # Isole par projet sur runner partagé
  paths: [node_modules/]
```

---

## Mesurer l'efficacité du cache

```yaml
.cache-metrics:
  after_script:
    - |
      echo "CACHE_HIT: $([ -d node_modules ] && echo 1 || echo 0)"
      echo "NODE_MODULES_SIZE: $(du -sh node_modules/ 2>/dev/null | cut -f1)"
```

Exposer via GitLab metrics → dashboard Grafana.
Target : > 80% hit rate sur `main`, > 60% sur les branches feature.
