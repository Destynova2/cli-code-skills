# Analogies -- Mental Models for Configuration Wizards

> **When to read:** When explaining wizard audit findings to stakeholders, or when seeking intuition for the right fix.
> These analogies map natural laws to specific wizard implementation rules.

---

## Universal law of configuration

> The more choices you offer upfront, the less likely the user is to finish -- or to make good choices.
> Complexity at setup time is entropy. Fight it with defaults and derivation.

---

## I. BIOLOGY -- Organism-level rules

### Progressive Config -> Seed Germination

A seed contains minimal genetic information but unfolds into a complex organism through environmental interaction (soil, light, water). The seed doesn't carry a blueprint for every leaf -- it carries rules that respond to context.

A good wizard asks for a **seed** (3-5 high-signal values) and derives the full configuration tree through defaults, detection, and convention. The OpenBSD installer is a seed: hit Enter through everything, get a viable system.

**Implication:** count your seed questions. If you're asking more than 5, you're encoding the whole tree in the seed instead of letting environment detection grow it.

### Progressive Disclosure -> Axolotl Facultative Metamorphosis

The axolotl is a salamander that reaches sexual maturity and reproduces without ever metamorphosing. It keeps its simple larval form (gills, aquatic body) indefinitely -- because it works. But when environmental stress demands it (thyroid hormone injection, habitat desiccation), the axolotl can undergo full metamorphosis into a terrestrial adult form. The genetic capacity for complexity exists but is suppressed until conditions demand it.

A wizard should behave like an axolotl: present the simple larval form (basic config) by default. The "terrestrial adult form" (advanced options, expert toggles, custom backends) exists in the genome but only surfaces when the user's environment demands it -- an explicit `--advanced` flag, a detected edge case, or a config key that signals expertise.

**Rule:** Never show the metamorphosed form by default. The axolotl lives its entire life as a larva in stable conditions. Your wizard should too. Advanced options are facultative -- triggered by environment, not imposed by design.

### Doctor Mode -> Immune System

| Biology | Config Doctor |
|---------|--------------|
| **Innate immunity** (immediate, non-specific) | Basic health checks: file exists, port open, service responds |
| **Adaptive immunity** (learned, specific) | Pattern-matching on known failure modes, historical fix database |
| **Antibodies** (tag for destruction) | Diagnostic checks that tag problems with fix recommendations |
| **Inflammation** (isolate damage) | Circuit breaker -- degrade gracefully, isolate broken config section |
| **Homeostasis** (maintain equilibrium) | Watchdog -- periodic health checks trigger corrective actions |
| **White blood cells** (repair agents) | Auto-repair scripts triggered by doctor findings |

The immune system doesn't need to understand the threat -- it recognizes "non-self" patterns. A config doctor should similarly detect anomalies without needing to understand every possible misconfiguration.

**Implication:** doctor checks should be pattern-based (schema validation, connectivity probes), not logic-based (trying to understand what the user intended).

### Credential Rotation -> MHC Molecule Turnover

Every nucleated cell in your body continuously displays peptide fragments on MHC class I molecules -- like a name badge that says "I'm self, I'm healthy." These badges are not permanent. MHC-peptide complexes have a half-life: on immature dendritic cells, MHC class II complexes turn over rapidly (hours), while on activated cells they can persist for 100+ hours. If a cell stops displaying valid badges, natural killer cells destroy it.

API tokens and credentials in a config should work exactly like MHC molecules:

| MHC Biology | Credential Management |
|-------------|----------------------|
| MHC displays peptides = "I am authorized" | Token proves service identity |
| Rapid turnover on immature cells | Short-lived tokens during setup/dev (hours) |
| Long half-life on activated cells | Longer-lived tokens in stable production (days) |
| Missing MHC = cell killed by NK cells | Expired token = service rejected by gateway |
| MHC restriction (only certain T-cells match) | Scoped tokens (read-only, namespace-limited) |

**Rule:** Wizard-generated tokens should have explicit TTLs. Default to short (dev = 1h, staging = 24h, production = 7d with rotation). Never generate a token without an expiry. A cell without an MHC badge gets killed -- a service without a valid token should too.

### Graceful Degradation -> Tardigrade Cryptobiosis

When conditions become extreme (desiccation, radiation, vacuum of space), tardigrades enter cryptobiosis: they retract their legs, form a "tun" (a protective ball), and reduce metabolism to 0.01% of normal -- or to zero. They can survive in this state for decades. When conditions improve, they rehydrate and resume full function within hours. Key molecular strategy: LEA proteins form a glass-like vitrification matrix that preserves cellular structure without active metabolism, and Dsup protein shields DNA from radiation damage.

A wizard should degrade like a tardigrade, not like a mammal:

| Tardigrade | Wizard |
|------------|--------|
| Forms a tun (protective shape) | Falls back to safe defaults when required services are unreachable |
| Reduces metabolism to 0.01% | Disables non-essential features, keeps core functional |
| LEA proteins preserve structure | Schema preserves config shape even with empty values |
| Rehydrates when conditions improve | Auto-resumes full function when dependencies come online |
| Survives vacuum, radiation, pressure | Works offline, with partial network, with stale caches |

**Rule:** Every config section should have a `tun state` -- the minimum viable configuration that keeps the service running when that section's dependencies are unavailable. Define it explicitly. Test it. The tardigrade does not panic when the water disappears; it has a plan.

### Idempotency -> Homeostatic Set Point

Body temperature is maintained at 37C through negative feedback. If you're cold, you shiver (generates heat). If you're hot, you sweat (dissipates heat). The key property: the system converges to 37C regardless of starting state. Run the thermoregulation process once or a thousand times -- you get 37C. It is naturally idempotent.

A wizard `apply` or `init` command must behave like thermoregulation:

**Rule:** `wizard apply` should be safe to run 1 time or 100 times. Like homeostasis, it should measure the current state, compare to the desired set point, and apply only the delta. If the system is already at 37C, applying heat would be harmful -- and a truly idempotent wizard does nothing when the config already matches the target.

```
# Idempotency test (homeostasis test):
wizard apply        # first run: changes state
wizard apply        # second run: no-op, exit 0
diff before after   # must be identical
```

### Migration / Versioning -> Epigenetics

DNA does not change between generations (barring mutation), but gene expression does -- through epigenetic modifications like DNA methylation. A methyl group attached to a gene silences it; remove the methyl group and the gene activates. The organism adapts its behavior across generations without rewriting its genome. These changes are reversible, heritable, and layered on top of the immutable base sequence.

Config versioning should work like epigenetics, not like mutation:

| Epigenetics | Config Versioning |
|-------------|-------------------|
| DNA sequence = immutable base | Core schema = stable across versions |
| Methylation = silence a gene | Feature flag = disable a config section |
| Demethylation = activate a gene | Feature flag = enable a new config section |
| Transgenerational inheritance | Config inherits from parent/default profile |
| Reversible without DNA damage | Version changes reversible without data loss |

**Rule:** Config schema is the genome -- it should change rarely and carefully. Behavioral changes between versions should be epigenetic: feature flags, overlays, profile inheritance. A v2 config should be a v1 config with different methylation (flags toggled), not a different genome (different schema).

### Defaults -> Tropism

Plants grow toward light (phototropism) and away from gravity (gravitropism) without being told. These are built-in defaults that work in 99% of environments.

Wizard defaults should be tropisms: `http_port: 80`, `tls: auto`, `log_level: info`. They work without thinking. The rare user who needs port 8443 knows they need it and will override.

**Implication:** if you're tempted to ask a question, ask: "would a plant ask this, or would it just grow toward the light?" If the answer is obvious for 90% of users, make it a default.

### Config Nesting -> Fractal Self-Similarity

Fern fronds, bronchial trees, and blood vessels repeat the same pattern at every scale. Config should too: project-level, user-level, and system-level config all follow the same schema shape.

**Implication:** if `project.toml` has `[backends]`, then `~/.app/config.toml` should too. Same keys, same structure, different scope. A user who understands one level understands all levels.

### Live Config Reload -> Homeostasis (Thermoregulation)

Body temperature stays at 37C through continuous monitoring and corrective feedback (sweating, shivering). Caddy's Admin API does the same: live config changes are validated, applied if safe, rolled back if not. No restart, no cold gap.

**Implication:** config changes should be hot-reloadable with validation. If the new config fails, the old one keeps running. Never leave the system in a broken state between old and new config.

### Regeneration -> Planarian Neoblasts

Cut a planarian flatworm into 279 pieces -- each piece regenerates into a complete organism. This is possible because planarians have neoblast stem cells distributed throughout their body: pluripotent cells that can differentiate into any of 40+ cell types. The organism does not need all its parts to function; any fragment contains enough information to rebuild the whole.

A wizard's config should be planarian-resilient:

**Rule:** Any subset of the config should be enough to bootstrap a working (if reduced) system. If the `[database]` section is missing, the wizard should still generate a valid config with an in-memory fallback. If `[auth]` is missing, default to a dev-mode no-auth stance. Each section is a neoblast -- it knows how to regenerate the defaults for its domain.

---

## II. PHYSICS -- Laws that map to wizard rules

### Minimum Questions -> Principle of Least Action (Lagrangian Mechanics)

In physics, the path a system takes between two states is the one that minimizes the action integral (kinetic energy minus potential energy, integrated over time). The system does not explore all possible paths -- it follows the one with minimum action. This is not a heuristic; it is a fundamental law of nature.

A wizard should follow the Principle of Least Action:

| Physics | Wizard |
|---------|--------|
| Start state -> end state | Blank slate -> working config |
| Minimize action integral | Minimize total questions asked |
| System follows optimal path naturally | Wizard derives maximum config from minimum input |
| Constraints reduce degrees of freedom | Environment detection eliminates questions |
| Stationary path = no unnecessary variation | No redundant or confirmatory questions |

**Rule: The Lagrangian Wizard Rule.** For every question the wizard asks, compute: "Could I derive this answer from (a) a previous answer, (b) environment detection, or (c) a safe default?" If yes to any, do not ask the question. The wizard's "action" (total user effort) must be minimized. Every unnecessary question is wasted energy -- a deviation from the optimal path.

### Config Drift -> Second Law of Thermodynamics (Entropy)

The second law states: in a closed system, entropy never decreases. Applied to software: Lehman's Second Law of Software Evolution says complexity increases unless active countervailing work is done. Configuration drift is thermodynamic entropy applied to infrastructure.

| Thermodynamics | Config Management |
|----------------|-------------------|
| Closed system entropy increases | Unmanaged config drifts from intended state |
| Energy required to decrease entropy | Engineering effort required to fight drift |
| Heat death = maximum entropy | Config chaos = every node is a snowflake |
| Refrigerator fights entropy with energy input | `wizard doctor` fights drift with periodic audits |
| Entropy is measurable (J/K) | Drift is measurable (diff lines from canonical) |

**Rule: The Entropy Tax.** Every config system needs a continuous energy input to fight drift. This is `wizard doctor --periodic`. Without it, the second law guarantees your configs will diverge. Budget for it. Measure drift as `diff` lines from canonical config. When drift exceeds a threshold, the doctor must trigger reconciliation -- like a refrigerator compressor kicking in.

### Config Migration -> Phase Transition

Water does not gradually become ice. At 0C, it undergoes a phase transition -- a discontinuous change in structure. The molecules are the same; their arrangement is fundamentally different. A phase transition requires latent energy (you must add or remove heat to cross the boundary), and during the transition, both phases can coexist.

Config v1 to v2 is a phase transition, not a gradual change:

| Phase Transition | Config Migration |
|-----------------|-----------------|
| Same molecules, different arrangement | Same values, different schema structure |
| Requires latent energy to cross boundary | Requires explicit migration script to cross versions |
| Both phases coexist during transition | Both config versions must be valid during rollout |
| Supercooling (delayed transition) | Deferred migration (old format still accepted) |
| Critical point (phases become indistinguishable) | Version compatibility layer (both formats map to same internal representation) |

**Rule:** Never assume migration is continuous. Design for discontinuity. Provide latent energy (migration scripts), support coexistence (both formats accepted during transition), and define the critical point (version after which old format is rejected).

### Conservation Laws -> Noether's Theorem

Emmy Noether proved: every continuous symmetry of a physical system corresponds to a conserved quantity. Time symmetry conserves energy. Spatial symmetry conserves momentum. Rotational symmetry conserves angular momentum. The deep insight: if a transformation leaves the system unchanged, something must be preserved.

In wizard config, define your symmetries and the corresponding conservation laws:

| Symmetry | Conserved Quantity | Wizard Rule |
|----------|-------------------|-------------|
| Time invariance (config works the same whenever applied) | Energy = service functionality | Applying the same config today or next month must produce the same running service |
| Translation invariance (config works across environments) | Momentum = portability | Config must produce equivalent results in dev, staging, prod (env-specific values parameterized, not hardcoded) |
| Rotation invariance (config works regardless of apply order) | Angular momentum = commutativity | Independent config sections must be order-independent |

**Rule: The Noether Audit.** For each config transformation (migration, override, merge), ask: "What symmetry is preserved? What must be conserved?" If a migration changes the schema but not the behavior, behavior is the conserved quantity -- test it. If a merge combines two configs, the union of their functionality must be conserved.

### Resonance -> Standing Waves

A guitar string vibrates at its resonant frequency when the driving force matches the string's natural frequency. At resonance, minimal energy produces maximum amplitude. Off-resonance, even large energy inputs produce weak response.

A wizard "resonates" when its mental model matches the user's mental model:

**Rule:** The wizard's question flow, terminology, and structure should match how the user thinks about the system -- not how the code is organized internally. If the user thinks "I need a database, a cache, and an API," the wizard should ask about database, cache, and API -- not about `persistence_layer`, `memoization_backend`, and `http_transport`. Resonance = minimum friction, maximum understanding. Test for resonance: if a user can predict the next question, the wizard is resonating.

---

## III. INFORMATION THEORY -- Optimal questioning

### Wizard Question Count -> Shannon Entropy Lower Bound

Shannon proved: the minimum average number of yes/no questions needed to determine a value from a probability distribution is the entropy H of that distribution. For a config wizard with N possible valid configurations, the theoretical minimum questions needed is log2(N). You cannot do better -- this is an information-theoretic floor.

**Rule: The Shannon Floor.** Count the number of meaningfully different configurations your wizard can produce. Take log2 of that number. That is the theoretical minimum number of binary choices needed. If your wizard asks significantly more questions than this, you are wasting the user's information bandwidth. Example: if there are 8 valid database configs (postgres/mysql/sqlite x local/remote, minus impossible combos), you need at most 3 binary questions -- not a 10-field form.

### Question Ordering -> Huffman Coding

Huffman coding assigns shorter codes to more probable symbols and longer codes to less probable ones. The optimal "twenty questions" strategy is a Huffman tree: ask the question whose answer eliminates the most possibilities first.

| Huffman Coding | Wizard Question Design |
|---------------|----------------------|
| Most probable symbol = shortest code | Most common config = fewest questions (just hit Enter) |
| Less probable = longer code | Exotic config = more questions (you chose this path) |
| Build tree bottom-up for optimality | Design question flow from common cases, not edge cases |
| Each bit maximally informative | Each question should halve the remaining config space |

**Rule: The Huffman Question Rule.** Order wizard questions by information gain -- the question that most reduces uncertainty about the final config goes first. Practically: ask "what type of project?" before "what port number?" because the project type eliminates entire branches of config. A question that does not eliminate at least one config branch is wasted.

### Doctor Mode Validation -> Error-Correcting Codes (Hamming Distance)

Hamming codes add redundant parity bits so that single-bit errors can be detected and corrected. The key concept: Hamming distance is the minimum number of bit flips needed to turn one valid codeword into another. If the minimum Hamming distance between valid configs is d, you can detect up to (d-1) errors and correct up to floor((d-1)/2) errors.

| Error-Correcting Codes | Config Doctor |
|------------------------|---------------|
| Valid codewords (small subset of all bit patterns) | Valid configs (small subset of all possible key-value combos) |
| Hamming distance between valid words | "Distance" between valid configs (number of fields that differ) |
| Parity bits = redundant checks | Doctor checks = redundant validations |
| Single-bit error correction | Single-field typo correction ("Did you mean `postgres`?") |
| Burst error detection | Correlated misconfiguration detection ("port 443 but tls: false") |

**Rule:** Design doctor checks to have Hamming distance >= 3 between valid configs -- meaning no single field change should silently produce a different valid-but-wrong config. If changing `database: postgres` to `database: postgre` is only 1 character away, the doctor should flag it as a likely typo rather than accepting it as a valid (but nonexistent) database type.

---

## IV. GAME DESIGN -- Teaching without a manual

### Wizard Flow -> Portal's 4-Step Teaching Pattern

Portal (Valve, 2007) teaches every mechanic through level design alone, with no tutorial text. The pattern:

1. **Isolation:** Introduce one mechanic in a safe, constrained room. No enemies, no timer, no other mechanics. The player can only interact with the new thing.
2. **Reinforcement:** A second room that requires the same mechanic but in a slightly different context. Confirms understanding.
3. **Combination:** A room that requires the new mechanic combined with a previously learned one. Tests integration.
4. **Mastery:** A room that requires creative application under pressure. Tests deep understanding.

Map to wizard design:

| Portal | Wizard |
|--------|--------|
| Step 1: Isolated safe room | Step 1: Ask one question at a time, with clear default, no side effects |
| Step 2: Reinforcement room | Step 2: Show the user what their answer produced (preview/dry-run) |
| Step 3: Combination room | Step 3: Show how this choice interacts with previous choices |
| Step 4: Mastery under pressure | Step 4: Final review before apply -- user sees full config, can edit any field |

**Rule: The Portal Rule.** Never combine new information with action. When introducing a concept (a new config section, a new option), show it in isolation first. Let the user see its effect before asking them to combine it with other choices. A wizard that asks 5 interdependent questions on one screen is a Portal level with 5 new mechanics at once -- it teaches nothing.

### Difficulty Scaling -> Celeste's Assist Mode

Celeste (Matt Thorson, 2018) is a brutally hard platformer that ships with a granular Assist Mode. The key insight from Thorson: "The most important assist options are the in-between ones, like slowing the game down 20%, or getting a single extra dash. Things that let the player fine-tune aspects of the difficulty, rather than make it trivial."

The mode was deliberately renamed from "Cheat Mode" to "Assist Mode" -- naming signals respect, not judgment.

| Celeste | Wizard |
|---------|--------|
| Assist Mode (non-judgmental name) | `--guided` or `--interactive` mode (not `--easy` or `--dumb`) |
| Slow game speed by 20% | Show explanations for each question |
| Extra dash (small capability boost) | Pre-fill smart defaults, let user just confirm |
| Invincibility (skip the challenge) | `--defaults` flag: accept all defaults, skip all questions |
| Fine-grained difficulty knobs | Per-section verbosity: `--explain=database` for one topic only |

**Rule: The Celeste Rule.** Provide granular assist, not binary easy/hard. A `--guided` flag that explains every question. A `--defaults` flag that skips everything. A `--explain=SECTION` flag for targeted help. Never make the user feel judged for needing help. Name your flags with respect.

### Save / Checkpoint -> Game Save Systems (Memento + Command Pattern)

Games use two patterns for saves: the **Memento pattern** (snapshot the entire state) and the **Command pattern** (record the sequence of actions, replay to restore). Each has tradeoffs:

| Game Pattern | Wizard Pattern |
|-------------|---------------|
| Memento: snapshot full state | `wizard snapshot` -- dump current config to versioned file |
| Command: record action sequence | `wizard history` -- log every config change as a reversible command |
| Checkpoint (auto-save at milestones) | Auto-snapshot before every `wizard apply` |
| Quick save / quick load | `wizard save myname` / `wizard load myname` |
| Undo stack (Command pattern) | `wizard undo` -- reverse last change, `wizard redo` -- reapply |

**Rule: The Checkpoint Rule.** Every destructive operation (`apply`, `migrate`, `reset`) must auto-create a checkpoint (Memento snapshot) before executing. The user should be able to `wizard undo` at any point. Never let the user lose their config to a bad migration. Games that delete your save file get 1-star reviews.

### Simple Rules, Emergent Config -> Conway's Game of Life

Conway's Game of Life has 4 rules: underpopulation (< 2 neighbors: die), survival (2-3 neighbors: live), overpopulation (> 3 neighbors: die), reproduction (exactly 3 neighbors: born). From these 4 rules, universal computation emerges. The system is Turing-complete.

A wizard's config grammar should aspire to this:

**Rule: The Conway Rule.** Define the fewest possible config primitives that compose into all needed configurations. If your config has 50 top-level keys, you don't have a grammar -- you have a dictionary. Aim for: a small set of composable primitives (service, backend, route, secret) that combine to express any deployment topology. Like Conway's 4 rules, the primitives should be simple enough to memorize, powerful enough to express anything.

Example primitives:
```
service:  name, port, replicas        -> defines a running thing
backend:  type, connection, pool_size -> defines a data store
route:    path, service, auth         -> defines connectivity
secret:   name, source, rotation      -> defines credentials
```
Four primitives. Compose them to describe any system.

---

## V. COGNITIVE SCIENCE / UX -- How humans process config

### Maximum Options Per Step -> Hick's Law

Hick's Law: decision time T increases logarithmically with the number of choices N: `T = a + b * log2(N+1)`. This is not a guideline -- it is a measured psychophysical law.

**Rule: The Hick's Law Limit.** No wizard step should present more than 5 choices. At 5 options, decision time is roughly `b * log2(6) = 2.6b`. At 20 options, it is `b * log2(21) = 4.4b` -- a 70% increase in decision time for the same question. If you must present more than 5, use hierarchical filtering: category first (3 choices), then option within category (3-5 choices). Two fast decisions beat one slow one.

### Maximum Info Per Screen -> Miller's Law (7 plus/minus 2)

George Miller (1956): the average person can hold 7 plus/minus 2 items in working memory simultaneously. More recent research suggests the true number is closer to 4 chunks for novel information.

**Rule: The Miller Screen Rule.** No wizard screen should display more than 5-7 config fields simultaneously. If a config section has 15 fields, chunk them: group related fields under labeled sub-headings (connection settings, pool settings, timeout settings). Each chunk becomes one "item" in working memory. The user processes 3 chunks of 5 fields, not 15 unrelated fields.

### Pre-Fill Values -> Recognition Over Recall

Nielsen's 6th heuristic: "Minimize the user's memory load by making elements, actions, and options visible." Recognition (seeing the right answer in a list) requires far less cognitive effort than recall (typing the right answer from memory). Uber pre-fills your likely destination based on time of day. Camera apps default to auto-mode.

**Rule: The Recognition Rule.** Every wizard field should pre-fill with the most likely value. Detection order: (1) environment variable, (2) previously saved value, (3) conventional default, (4) empty with placeholder showing example format. The user should be able to complete the wizard by pressing Enter on every field and get a working config. Recall is expensive; recognition is cheap. Pre-fill everything.

### Question Chunking -> Cognitive Load Theory

Cognitive Load Theory (Sweller, 1988) distinguishes intrinsic load (inherent complexity of the material), extraneous load (poor presentation that wastes working memory), and germane load (useful mental effort that builds understanding). A wizard should minimize extraneous load and maximize germane load.

**Rule: The Load Audit.** For each wizard screen, classify every element:
- **Intrinsic:** the actual config choices (unavoidable, minimize by reducing questions)
- **Extraneous:** decorative UI, redundant labels, unexplained jargon (eliminate)
- **Germane:** helpful context, preview of what the choice does, examples (keep and improve)

If a wizard screen has more extraneous elements than germane ones, redesign it.

---

## Summary table

| # | Wizard Concept | Domain | Analogy | Key Rule |
|---|---------------|--------|---------|----------|
| 1 | Progressive config | Biology | Seed germination | Minimal input + environment = full config |
| 2 | Progressive disclosure | Biology | Axolotl facultative metamorphosis | Never show advanced form unless environment demands it |
| 3 | Doctor mode | Biology | Immune system | Detect anomalies by pattern, not by logic |
| 4 | Credential rotation | Biology | MHC molecule turnover | Tokens must have TTLs; no badge = killed |
| 5 | Graceful degradation | Biology | Tardigrade cryptobiosis | Every section needs a tun state (minimum viable config) |
| 6 | Idempotency | Biology | Homeostatic set point (37C) | Apply measures delta, converges to target, safe to repeat |
| 7 | Migration / versioning | Biology | Epigenetics (DNA methylation) | Schema is genome (stable); behavior changes are flags (reversible) |
| 8 | Defaults | Biology | Plant tropism | Grow toward the obvious, don't ask |
| 9 | Config nesting | Biology | Fractal self-similarity | Same shape at every scope level |
| 10 | Live reload | Biology | Thermoregulation homeostasis | Continuous correction, never cold gap |
| 11 | Regeneration | Biology | Planarian neoblasts | Any config fragment can rebuild defaults for its domain |
| 12 | Minimum questions | Physics | Principle of least action | Derive what you can; only ask what you must |
| 13 | Config drift | Physics | Second law of thermodynamics | Entropy increases without continuous energy input (doctor) |
| 14 | Config migration | Physics | Phase transition | Discontinuous change; support coexistence during transition |
| 15 | Invariants | Physics | Noether's theorem | Every symmetry implies a conserved quantity; test it |
| 16 | Mental model match | Physics | Resonance / standing waves | Wizard resonates when it matches user's natural frequency |
| 17 | Question count floor | Info Theory | Shannon entropy | log2(N valid configs) = minimum binary questions |
| 18 | Question ordering | Info Theory | Huffman coding | Most informative question first; common path = shortest |
| 19 | Doctor validation | Info Theory | Hamming distance / ECC | Single-field typos detectable and correctable |
| 20 | Wizard flow | Game Design | Portal 4-step teaching | Isolate, reinforce, combine, master |
| 21 | Difficulty scaling | Game Design | Celeste Assist Mode | Granular assist, non-judgmental naming |
| 22 | Save / undo | Game Design | Memento + Command pattern | Auto-checkpoint before every destructive op |
| 23 | Config grammar | Game Design | Conway's Game of Life | Fewest primitives that compose into all configs |
| 24 | Max options/step | Cognitive Science | Hick's Law | No more than 5 choices per wizard step |
| 25 | Max fields/screen | Cognitive Science | Miller's Law (7 +/- 2) | Chunk into 3-5 groups, max 7 fields visible |
| 26 | Pre-fill values | Cognitive Science | Recognition over recall | Pre-fill everything; Enter-through = working config |
| 27 | Screen design | Cognitive Science | Cognitive Load Theory | Minimize extraneous load, maximize germane load |

---

## Sources

### Biology
- [Experimentally induced metamorphosis in axolotls (PMC)](https://pmc.ncbi.nlm.nih.gov/articles/PMC4895291/)
- [Rediscovering the Axolotl as a Model for Thyroid Hormone Dependent Development (Frontiers)](https://www.frontiersin.org/journals/endocrinology/articles/10.3389/fendo.2019.00237/full)
- [MHC class I antigen presentation pathway (PMC)](https://pmc.ncbi.nlm.nih.gov/articles/PMC1783040/)
- [Major Histocompatibility Complex and its functions (NCBI)](https://www.ncbi.nlm.nih.gov/books/NBK27156/)
- [Cryptobiosis protects from extremes (AskNature)](https://asknature.org/strategy/cryptobiosis-protects-from-extremes/)
- [Tardigrade research on extreme survival mechanisms (The Scientist)](https://www.the-scientist.com/tardigrade-research-on-extreme-survival-mechanisms-73816)
- [Planarian regeneration cellular and molecular basis (PMC)](https://pmc.ncbi.nlm.nih.gov/articles/PMC7706840/)
- [Homeostasis (Khan Academy)](https://www.khanacademy.org/science/biology/principles-of-physiology/body-structure-and-homeostasis/a/homeostasis)
- [Epigenetics (Wikipedia)](https://en.wikipedia.org/wiki/Epigenetics)
- [Role of Methylation in Gene Expression (Nature Scitable)](https://www.nature.com/scitable/topicpage/the-role-of-methylation-in-gene-expression-1070/)

### Physics
- [Principle of Least Action (Feynman Lectures)](https://www.feynmanlectures.caltech.edu/II_19.html)
- [Principle of Least Action (Scholarpedia)](http://www.scholarpedia.org/article/Principle_of_least_action)
- [Software Entropy and Thermodynamics (Java Code Geeks)](https://www.javacodegeeks.com/2026/03/the-thermodynamics-of-software-entropy-why-all-code-tends-toward-disorder.html)
- [Phase transition (Wikipedia)](https://en.wikipedia.org/wiki/Phase_transition)
- [Noether's theorem (Wikipedia)](https://en.wikipedia.org/wiki/Noether's_theorem)
- [Noether's Theorem: Complete Guide (Profound Physics)](https://profoundphysics.com/noethers-theorem-a-complete-guide/)

### Information Theory
- [How Shannon's Entropy Quantifies Information (Quanta Magazine)](https://www.quantamagazine.org/how-claude-shannons-concept-of-entropy-quantifies-information-20220906/)
- [Twenty Questions and Shannon Entropy (arXiv)](https://arxiv.org/abs/1611.01655)
- [Shannon Entropy and Information Gain (Udacity / Medium)](https://medium.com/udacity/shannon-entropy-information-gain-and-picking-balls-from-buckets-5810d35d54b4)
- [Huffman coding (Wikipedia)](https://en.wikipedia.org/wiki/Huffman_coding)
- [Hamming code (Wikipedia)](https://en.wikipedia.org/wiki/Hamming_code)

### Game Design
- [Portal level design: a game that teaches (Battzcave)](https://battzcave.wordpress.com/2016/05/14/leveldesignofvideogames06-portal/)
- [Learn game design principles from Portal developers (freeCodeCamp)](https://www.freecodecamp.org/news/learn-game-design-principles-from-valve-portal-developers/)
- [Celeste Assist Mode change (Vice)](https://www.vice.com/en/article/celeste-assist-mode-change-and-accessibility/)
- [Celeste: breaking rules with Assist Mode (Vice)](https://www.vice.com/en/article/celeste-difficulty-assist-mode/)
- [Command pattern in game programming (Game Programming Patterns)](https://gameprogrammingpatterns.com/command.html)
- [Conway's Game of Life: simple rules, extraordinary complexity (Medium)](https://medium.com/@thefirebearerblog_30745/conways-game-of-life-how-simple-rules-create-extraordinary-complexity-900730c94fec)
- [GDC Vault: Breath of the Wild conventions talk](https://www.gdcvault.com/play/1024562/Change-and-Constant-Breaking-Conventions)
- [Dark Souls: the IKEA of Games (80lv)](https://80.lv/articles/gdc-talk-why-dark-souls-is-the-ikea-of-games)

### Cognitive Science / UX
- [Miller's Law (Laws of UX)](https://lawsofux.com/millers-law/)
- [Miller's Law in UX: why 7 +/- 2 isn't just math (Designzig)](https://designzig.com/millers-law-in-ux-design/)
- [Recognition vs Recall (IxDF)](https://ixdf.org/literature/topics/recognition-vs-recall)
- [Recognition over Recall UX principle (Medium)](https://medium.com/@digitalofthings/ux-principle-6-recognition-over-recall-620d284436b3)
- [Tried and True Laws of UX (Toptal)](https://www.toptal.com/designers/ux/laws-of-ux-infographic)
