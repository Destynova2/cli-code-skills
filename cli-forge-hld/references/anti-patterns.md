# Anti-Patterns to Avoid

> **When to read:** During review, to check the HLD doesn't fall into common traps.

---

| Anti-Pattern | Description | Fix |
|-------------|-------------|-----|
| **The Novel** | 50+ page doc nobody reads | Keep to 10-20 pages, use appendices |
| **Architecture Astronaut** | Over-abstraction, buzzwords, no concrete decisions | Ground every section with specific tech choices |
| **The Wedding Cake** | Pretty layered diagram with no protocol/data info | Label every arrow with protocol + format |
| **Missing Non-Goals** | Scope creep because boundaries aren't explicit | Always include Non-Goals section |
| **Decision Amnesia** | No record of why choices were made | ADR for every significant decision |
| **Golden Hammer** | Same tech for everything regardless of fit | Evaluate with weighted decision matrix |
| **Ivory Tower** | Architect designs in isolation | Collaborative drafting + review (Google model) |
| **Premature Scale** | Over-engineering for traffic not yet seen | Design for current + 10x, not 1000x |
| **Copy-Paste Schema** | Dumping full DB schemas into HLD | Sketch entities only; column detail is LLD |
| **Numbers-Free Design** | No capacity estimation, just gut feelings | Always run back-of-envelope before choosing infra |
