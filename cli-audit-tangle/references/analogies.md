# Analogies — Mental Models for Structural Tangling

> **When to read:** When explaining findings to stakeholders, or when seeking intuition for the right fix.
> These analogies map physical/biological laws to code topology problems.

---

## Universal law of entanglement

> The longer, thinner, and more flexible a system is, the more entropy pushes it toward entanglement.
> Untangling costs energy — or intelligent geometry.

True for hair, spaghetti, DNA, earphone cables... and code.

---

## By anti-pattern

### Circular Dependency → Topological knot (Anti-Pattern #2)

A true knot cannot be undone without passing a free end through it.
In code: a hard cycle cannot be resolved without extracting an interface or a third module.
No shortcut — you must cut and pass through an intermediary.

**Implication:** identify the free end (the cheapest dependency to move) and extract it.

### God Function → Fire ant overload (Anti-Pattern #1)

Fire ants form living rafts where each ant grips exactly 2 neighbors.
If one ant grips 12, the raft collapses at that point.
A god function grips too many neighbors — the codebase is fragile there.

**Implication:** reduce the grip count. Extract sub-functions until each has ≤ 2-3 responsibilities.

### Strong Coupling → Jellyfish tentacles (Anti-Pattern #3, #4)

Jellyfish tentacles don't tangle thanks to mucus (reduced friction) and constant flow (reset).
Strong coupling = two modules without mucus — they stick on every contact.

**Fix:** introduce an interface (the mucus). Each module only knows the contract, not the implementation.

### God Module / Hub → Solar eruption (Anti-Pattern #6)

In the Sun, magnetic field lines progressively tangle around a single point.
When that point gives way, stored energy explodes as a solar eruption.
A god module works the same way — everything depends on it, and when it changes, everything explodes.

**Fix:** progressively discharge the god module by extracting cohesive sub-domains.
Never refactor everything at once — that IS the solar eruption.

### CI/CD Deadlock → Railway impasse (Pipeline anti-pattern)

Two trains on a single track facing each other. Neither can advance.
In CI: job A waits for job B (`needs: [B]`), job B waits for job A (`needs: [A]`). Deadlock.

**Fix:** identify which job can be split in two (a "switch point").
Often, a job does too many things — extract the common part into an upstream job with no dependencies.

### Hidden Transitive Dependencies → DNA and topoisomerases (Anti-Pattern #7)

2 meters of DNA in 6 micrometers. It tangles constantly during replication.
Topoisomerases cut one strand, pass it over the other, and re-ligate.
An active untangler — it doesn't prevent tangling, it repairs it continuously.

In code: hidden transitive dependencies (A → B → C but A doesn't know C exists)
are rarely detected before causing a problem. You need an active tool (linter, dep-check) running in CI.

**Fix:** no single fix — set up a `dependency-check` in CI that plays the topoisomerase role.

### Progressive Refactoring → Untangling hair

Always start from the tips, never from the root.
Pulling from the root compacts the knot (increases pressure, therefore friction).
Starting from the tips frees loops one by one.

In code: start with graph leaves (modules without outgoing dependencies), never the core.
Extract utils, types, pure interfaces first. The core untangles naturally once the tips are free.

### Codebase Viscosity → Cooked spaghetti / Polymer

A polymer's viscosity (rubber, melted plastic) directly depends on chain entanglement.
Longer, more entangled chains = more viscous system = slow, hard to modify.

In code: a highly coupled codebase is viscous. Every modification requires touching 10 files.
Viscosity is measurable: files touched per PR, build time, test coverage.

**Fix:** reduce average dependency chain length (graph depth). Target: no module deeper than 3 levels from entry.

---

## Summary table

| Code/CI Problem | Physical Analogy | Key Fix |
|-----------------|-----------------|---------|
| Hard cycle (ImportError) | Topological knot | Extract interface/third module |
| Strong coupling | Tentacles without mucus | Introduce an interface |
| God module | Solar magnetic field | Progressive extraction |
| CI deadlock | Railway impasse | Split a job in two |
| Transitive dependencies | DNA during replication | dep-check in CI (topoisomerase) |
| Progressive refactoring | Untangling hair | Start from leaves, not the core |
| Viscous codebase | Entangled polymer | Reduce graph depth |
