# Slide Structure & Narrative Frameworks

> **When to read:** Step 2 (Choose narrative framework) and Step 3 (Generate slides).

---

## The 5-second rule (the peacock test)

Every slide must pass this test: **can the audience grasp the message in 5 seconds?**

If not, the slide is either too dense (too many words, too much code) or the message is unclear (label heading instead of assertion heading). A peacock ocelle is recognized from across the clearing in one glance — your slide must do the same from the back of the room.

**Operational checks:**
- Is there one (and only one) message on this slide?
- Would removing half the text improve clarity?
- Is the heading an assertion or a label?

---

## The Assertion-Evidence model (Michael Alley, Penn State)

The single most impactful change you can make to any slide deck:

**Heading = full-sentence assertion** (what you want the audience to remember).
**Body = visual evidence** supporting that assertion.

| Bad (label heading) | Good (assertion heading) |
|---------------------|--------------------------|
| `# Performance` | `# Cold starts dropped from 12s to 800ms` |
| `# Architecture` | `# The proxy sits between the client and the LLM provider` |
| `# Security` | `# Every request is signed with ECDSA-P256 before forwarding` |
| `# Results` | `# 99.7% of production incidents are caught by the contract checker` |

**Why it works:** retention studies show that audiences remember assertion headings 2-3x better than label headings. The heading IS the takeaway.

---

## Named structural patterns

### Progressive Disclosure

Start with a simplified diagram. Each subsequent slide adds one element to the same diagram. By the final slide, the full architecture is visible — but the audience built the mental model incrementally, not all at once.

**Marp technique:** use the same `![bg]()` background across 3-4 slides, each with an overlay that adds one element. Or use Mermaid diagrams that grow.

### Contrast Pair (Before/After)

Two slides back-to-back: identical layout, different content. "Before" shows the painful state. "After" shows the improvement. The visual parallel makes the delta visceral.

```
Slide N:   "Before: 45-minute deploys, 2 outages/month"
Slide N+1: "After: 3-minute deploys, 0 outages in 6 months"
```

Same font, same layout, same position. Only the data changes.

### Terminal Typewriter

Instead of showing a screenshot of terminal output (which is dense and overwhelming), build it up:

- **Slide 1:** the command typed
- **Slide 2:** command + first lines of output
- **Slide 3:** the key output line highlighted

This forces the audience to read at speaking pace, not scanning pace.

### The Live Coding Sandwich

1. **Slides** explaining the concept (bread)
2. **Live demo** / live coding (filling)
3. **Slides** summarizing what just happened (bread)

Never do live coding without the bread — the audience needs the mental model before and the reinforcement after.

### Callback Close

Reference your opening hook in your final slide. If you opened with a problem ("45-minute deploys"), close by showing that problem solved ("3-minute deploys — and here's how you can get there too"). Creates narrative closure. The pheromone trail loops back to the nest.

### Zoom In / Zoom Out

1. Start at the highest altitude (industry trend, macro problem)
2. Zoom into your specific case
3. Deep technical dive
4. Zoom back out to show how this connects to the bigger picture

Gives the audience a sense of significance — "this isn't just our problem, it's the industry's problem."

---

## Narrative framework reference

### SCR — Situation, Complication, Resolution (Minto Pyramid)

Origin: Barbara Minto, McKinsey. The gold standard for technical talks.

- **Situation**: "We run 200 microservices on Kubernetes."
- **Complication**: "Deploying took 45 minutes and broke prod twice a month."
- **Resolution**: "We built X, and here's how it works."

**Key insight:** state the resolution EARLY (within the first 2 minutes). Then spend the talk proving it. Don't save the punchline for the end — the audience will check their phone if you do.

### 5-Box — What, Why, How, What If, What Next

Common at KubeCon/GopherCon:

1. **What** is the thing? (30 sec)
2. **Why** should anyone care? (2 min)
3. **How** does it work? (bulk)
4. **What If** — edge cases, failure modes (builds credibility)
5. **What Next** — CTA, roadmap

### PMR — Problem, Mechanism, Result

Engineering-flavored SCR:

1. **Problem**: precisely define what's broken (metrics, error rates)
2. **Mechanism**: how does the solution work, mechanically?
3. **Result**: before/after dashboard

### Boring Problems Are Interesting (KubeCon SIG style)

1. "Here's a problem you think is solved"
2. "It's actually not — here's the evidence"
3. "Here's what we tried that didn't work" (builds trust)
4. "Here's what worked and why"
5. "Here's what we still haven't figured out" (honesty as credibility)

### Pixar Pitch (adapted for tech)

- Once upon a time, we had [system].
- Every day, we [routine operation].
- One day, [incident / scaling event / new requirement].
- Because of that, we [first attempt].
- Because of that, we [learned / pivoted].
- Until finally, we [built the solution].

### Inverted Pyramid (newspaper style)

State the conclusion first. Then layers of increasing detail. If the audience zones out at minute 15, they already have the key message.

Best for internal tech talks and sprint reviews where decision-makers want the answer, not the journey.

---

## Minimalist approaches (optional modes)

### Takahashi Method

- **Huge text** — one word or very short phrase per slide
- **No images** — text only
- **Fast pace** — 2-5 seconds per slide, hundreds of slides for 20 min
- Slides are a rhythm device, not content

Use `<!-- fit -->` in Marp headings for auto-scaling.

### Lessig Method

Like Takahashi but alternates text-only and full-bleed image-only slides. Each slide is one "beat" in the argument.

### Kawasaki 10/20/30

- **10 slides** maximum
- **20 minutes** maximum
- **30pt font** minimum

The font rule is the one that matters most — it forces brevity.
