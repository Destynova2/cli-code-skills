# Anti-Patterns to Avoid

> **When to read:** Load this file during LLD review to check for common design mistakes. Contains 9 anti-patterns with descriptions and fixes.

---

| Anti-Pattern | Description | Fix |
|-------------|-------------|-----|
| **God Class** | One class doing everything | Apply SRP — split into focused classes |
| **Deep Inheritance** | 4+ levels of inheritance | Prefer composition over inheritance |
| **Over-Patterned** | Strategy for one algorithm, factory for two types | YAGNI — patterns when the problem demands them |
| **Happy Path Only** | No error handling, no concurrency analysis | Ask "What if this fails?" and "What if two things happen simultaneously?" |
| **Stale LLD** | Written once, abandoned | Store near code, review in PRs, treat as living doc |
| **No Error Taxonomy** | Errors as generic strings | Define error codes, categories, handling strategies |
| **Invisible Dependencies** | Constructor hides what it needs | Explicit DI via constructor parameters |
| **Anemic Domain** | Domain objects are just data bags | Move business rules into domain entities |
| **Copy-Paste from HLD** | Repeating HLD content instead of zooming in | Reference HLD, then go deeper |
