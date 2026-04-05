# Fix Patterns — Concrete Remediation by Domain

> **When to read:** After Step 4 classification, to provide actionable fix recommendations.
> Each pattern maps to a specific anti-pattern from `anti-patterns.md`.

---

## Rust

### Circular Dependencies (Anti-Pattern #2)

Cargo prevents cycles between crates at compile time. Within a single crate, modules form a strict tree — no import cycles possible.

**Pattern 1: Extract shared crate**
```toml
# Before: crate-a depends on crate-b, crate-b depends on crate-a → IMPOSSIBLE

# After: extract crate-common
# crate-a depends on crate-common
# crate-b depends on crate-common
```

**Pattern 2: Trait-based dependency inversion**
```rust
// Before: module A calls B directly, B calls A directly

// After: A defines a trait, B implements it
// a/mod.rs
pub trait Processor {
    fn process(&self, input: &str) -> Result<Output>;
}

// b/mod.rs
impl a::Processor for BProcessor { ... }
```

**Pattern 3: Feature flags for optional coupling**
```toml
[dependencies]
crate-b = { version = "1.0", optional = true }

[features]
with-b = ["crate-b"]
```

### God Function (Anti-Pattern #1)

**Pattern: Extract by responsibility**
```rust
// Before: one 300-line function doing parse + validate + transform + persist

// After:
fn process(input: &str) -> Result<Output> {
    let parsed = parse(input)?;
    let validated = validate(parsed)?;
    let transformed = transform(validated)?;
    persist(transformed)
}
```

Each extracted function has a single responsibility and can be tested independently.

### Hub and Spoke (Anti-Pattern #6)

**Pattern: Introduce domain modules**
```
# Before: src/core.rs has 50% of all inter-module edges

# After:
src/core/routing.rs    → handles routing logic
src/core/dispatch.rs   → handles dispatch logic
src/core/transform.rs  → handles transformation
```

Split the hub by domain, not by arbitrary size. Each sub-module should have high cohesion.

---

## Python

### Cycle d'import (Anti-Pattern #2 — cycle dur)

**Symptom**
```
ImportError: cannot import name 'Foo' from partially initialized module 'bar'
```

**Pattern 1: Extract to a third module**
```
Before: A imports B, B imports A
After:  A imports C, B imports C  (C = extracted shared types)
```
Extract shared types, constants, or interfaces into `common.py`, `types.py`, or `interfaces.py`.

**Pattern 2: Lazy import**
```python
# Before (top-level, creates cycle)
from bar import Foo

# After (import inside function)
def some_function():
    from bar import Foo  # only imported at call time
    return Foo()
```
Quick fix but technical debt — use to unblock, not as final solution.

**Pattern 3: TYPE_CHECKING guard**
```python
from __future__ import annotations
from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from bar import Foo  # not evaluated at runtime

def process(item: "Foo") -> None:  # string annotation
    ...
```
Ideal when the cycle is only due to static typing.

---

## TypeScript / Node

### Cycle d'import (Anti-Pattern #2)

**Pattern 1: Shared types module**
```
Before: feature-a/index.ts imports feature-b/service.ts
        feature-b/service.ts imports feature-a/types.ts

After:  shared/types.ts contains common types
        feature-a and feature-b import from shared/
```

**Pattern 2: Dependency Inversion**
```typescript
// Before — direct coupling
class OrderService {
  constructor(private userService: UserService) {}
}

// After — interface coupling
interface IUserService {
  getUser(id: string): User;
}
class OrderService {
  constructor(private userService: IUserService) {}
}
```

**Detection in CI**
```bash
npx depcruise src --include-only "^src" --output-type err
# Returns exit code 1 if cycle detected — integrate in CI
```

---

## CI/CD — GitHub Actions

### Workflow cycle (Anti-Pattern #2 applied to CI)

**Symptom:** workflow A triggers workflow B which triggers workflow A.

**Pattern: Central orchestrator**
```yaml
# Before: on: workflow_run creates a cycle

# After: use a central "orchestrator" workflow
# orchestrator.yml calls A and B explicitly
# A and B never trigger each other
```

**Detection**
```bash
grep -r "workflow_run\|workflow_call" .github/workflows/ | \
  awk -F: '{print $1, $0}' | sort
```

### `needs` deadlock

**Symptom:** job stuck "waiting", runners available but nothing runs.

**Pattern 1: Upstream common job**
```yaml
# Before: build-a needs test-b, test-b needs build-a → deadlock

# After: extract prepare with no needs
prepare:
  stage: .pre
  script: ./scripts/prepare.sh

build-a:
  needs: [prepare]

test-b:
  needs: [prepare]
```

**Pattern 2: Explicit DAG**
```yaml
# Each `needs` must point to a job in a PREVIOUS stage ONLY
# Draw the graph before writing the YAML
```

**Detection**
```bash
grep -r "needs:" .github/workflows/ | grep -v "^#"
# Build the graph manually — any cycle = deadlock
```

---

## CI/CD — GitLab

### `needs` deadlock

**Diagnostic**
```bash
python3 -c "
import yaml
with open('.gitlab-ci.yml') as f:
    ci = yaml.safe_load(f)
for job, config in ci.items():
    if isinstance(config, dict) and 'needs' in config:
        needs = [n if isinstance(n, str) else n.get('job','?') for n in config['needs']]
        print(f'{job} -> {needs}')
"
```

**Pattern: Enforce stage ordering**
```yaml
stages:
  - prepare    # no dependencies
  - build      # depends prepare
  - test       # depends build
  - deploy     # depends test
```
Stages are a structural guard against cycles — enforce their use.

---

## General Architectural Patterns

### Event Bus (decouples Anti-Pattern #2, #6)
```
Before: A calls B directly
After:  A publishes an event, B subscribes
```
A and B no longer know each other. Ideal for God modules / Hub and Spoke.
Cost: harder to debug (non-linear flow).

### Interface Segregation (fixes Anti-Pattern #4 — Feature Envy)
```
Before: A depends on B (200 methods)
After:  A depends on IFooable (3 methods A actually uses)
```
Reduce the contact surface between modules.

### Strangler Fig (progressive refactoring)
```
1. Create the new clean module alongside the old one
2. Progressively route traffic to the new module
3. When the old one has zero traffic, remove it
```
Start from the leaves. Never refactor the core in one shot.

---

## Validation Metrics

After applying a fix, verify with these targets:

| Metric | Before (expected) | After (target) |
|--------|-------------------|----------------|
| Cycles in graph | N > 0 | 0 |
| Max graph depth | > 5 | ≤ 3 |
| God module Ca (afferent coupling) | > 20 | < 5 per module |
| Build time | baseline | -20% min |
| Files touched per PR (avg) | > 10 | < 5 |
| CI jobs blocked | > 0 | 0 |
