# Cache Strategy — Slime Mold + Mycelium Combined

## Principle: the cache is the colony's nervous system

Without shared memory, every ant starts from zero.
The CI cache = pheromones and mycelium combined.

---

## Cache levels (mycelium waterfall)

```
Job on feature/xyz
  └─ Look up cache key: deps-feature-xyz         → MISS
      └─ Look up cache key: deps-main             → HIT ✓ (partial restore)
          └─ Otherwise pre-built Docker image     → MISS
              └─ Otherwise full rebuild           → SLOW PATH
```

### GitLab CI — Cache waterfall

```yaml
cache:
  - key:
      files: [package-lock.json]
      prefix: node-$CI_COMMIT_REF_SLUG
    paths: [node_modules/]
    policy: pull-push          # Current branch: reads + writes

  - key:
      files: [package-lock.json]
      prefix: node-main
    paths: [node_modules/]
    policy: pull               # Main fallback: reads only

  - key: node-base-image
    paths: [node_modules/]
    policy: pull               # Ultimate fallback: reads only
```

---

## Cache key strategies per ecosystem

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

### Docker layers (Physarum: reinforce the tubes that are used)
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

## Physarum strategy: adaptive cache by change profile

The slime mold reinforces tubes that are used and prunes the others.

```yaml
# Jobs touched by frontend only
.frontend-cache:
  cache:
    key:
      files: [frontend/package-lock.json]
      prefix: frontend
    paths: [frontend/node_modules/]

# Jobs touched by backend only
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

## Common pitfalls

### ❌ Cache key too broad → cache useless
```yaml
# BAD: invalidated on every commit
cache:
  key: "$CI_COMMIT_SHA"   # Never a hit

# BAD: invalidated on every branch
cache:
  key: "$CI_COMMIT_REF_NAME"  # Ephemeral branches = ephemeral cache
```

### ❌ Non-scoped cache → cross-project pollution
```yaml
# BAD on a shared runner
cache:
  paths: [node_modules/]
  # Default key = $CI_JOB_NAME, OK but risks collision
```

### ✅ Scoped and content-based cache
```yaml
cache:
  key:
    files: [package-lock.json]
    prefix: $CI_PROJECT_PATH_SLUG  # Isolate per project on a shared runner
  paths: [node_modules/]
```

---

## Measuring cache effectiveness

```yaml
.cache-metrics:
  after_script:
    - |
      echo "CACHE_HIT: $([ -d node_modules ] && echo 1 || echo 0)"
      echo "NODE_MODULES_SIZE: $(du -sh node_modules/ 2>/dev/null | cut -f1)"
```

Expose via GitLab metrics → Grafana dashboard.
Target: > 80% hit rate on `main`, > 60% on feature branches.
