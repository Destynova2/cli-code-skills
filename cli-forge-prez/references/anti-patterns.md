# Presentation Anti-Patterns

> **When to read:** Step 3 (Generate) and Step 5 (Quality checklist). Avoid these during generation; flag them if they exist in input material.

---

## 11 Anti-Patterns

| Anti-Pattern | Detection | Fix |
|---|---|---|
| **Feature List** | Slide with > 5 bullet points listing features/capabilities | One slide = one message. A feature list is a backlog, not a presentation. Pick the 1-2 features that matter for this audience |
| **Wall of Code** | Code block > 8 lines on a slide | Extract the 2-3 lines that carry the point. Put context in speaker notes. Use `<!-- fit -->` or reduce non-key lines to comments |
| **Architecture Astronaut** | Full system diagram with 15+ boxes on slide 3, before the audience knows what the project does | Show the simplest possible version first (3 boxes). Use Progressive Disclosure to add complexity across slides |
| **Demo Gods** | Live demo without fallback — one timeout or typo and the talk dies | Always have a pre-recorded terminal session (asciinema) or screenshots. The Live Coding Sandwich pattern: slides → demo → slides |
| **Slideument** | Slides readable as a standalone document without a speaker | The slides support the speaker, they don't replace them. If you can email the deck and it makes sense, it's a document — rewrite as slides |
| **Curse of Knowledge** | Jargon or acronyms used in the first 3 slides without definition | If the term isn't common knowledge for this audience, define it or eliminate it. One undefined term = one lost listener |
| **Apology Slide** | "Sorry this is hard to read", "I know this is a lot of text" | If you're apologizing, the slide is wrong. Delete and rewrite. Never show something you know doesn't work |
| **Bibliography** | Final slide with 15+ URLs nobody will type | One QR code linking to a page with all resources. Or one short URL. The audience will photograph it, not type it |
| **Outline Slide** | "Today I'll cover: 1) Background, 2) Architecture, 3) Results..." | Jump straight into the hook. Outline slides waste 30 seconds and the audience ignores them. The structure should be felt, not announced |
| **Font Roulette** | 4+ different fonts or sizes on a single slide | One font family, three sizes maximum (heading, body, code). Consistency is a signal of craft |
| **Premature Abstraction** | Explaining the generic pattern before showing the concrete example | Audiences understand concrete-to-abstract, never abstract-to-concrete. Show the specific case first, then generalize |

## Red Flags During Generation

These are signals that the generated deck is drifting into anti-pattern territory:

- **More than 2 bullet points start with "Support for..."** → Feature List
- **A code block wraps to more than 8 visible lines** → Wall of Code
- **The word "architecture" appears before the word "problem"** → Architecture Astronaut
- **No speaker notes on a slide** → potential Slideument
- **A heading is a single noun** (`# Security`, `# Performance`) → needs Assertion-Evidence rewrite
- **The last slide has more than 3 links** → Bibliography
- **Slide 2 is "Agenda" or "Outline"** → Outline Slide

## The Pheromone Trail Test

After generating the full deck, run this check for every slide:

1. **Remove the slide mentally**
2. Does the narrative trail break? (audience would be confused by the next slide)
   - **Yes** → the slide is load-bearing. Keep it.
   - **No** → the slide is noise. Delete it.

Every deck should be the minimum number of slides that maintains a continuous pheromone trail from hook to CTA. No filler, no "nice to have" slides.
