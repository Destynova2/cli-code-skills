# Presentation Tiers

> **When to read:** Step 0 (Determine tier) and Step 2 (Choose narrative framework).

---

## Tier: pitch (3-4 slides, ~2 min)

**Context:** Elevator pitch, README hero section as slides, hallway conversation at a conference, tweet-thread visual version.

**Default structure — Hook → Solution → Proof → CTA:**

| Slide | Purpose | Content | Duration |
|-------|---------|---------|----------|
| 1 | **Hook + Problem** | One provocative sentence or stat. The pain point. | 20 sec |
| 2 | **Solution + Visual** | What the project does, in one sentence. One diagram or screenshot. No feature list. | 40 sec |
| 3 | **Proof** | Numbers, benchmarks, adoption, or a live 15-second demo result. | 30 sec |
| 4 | **CTA** | Repo URL (or QR code), one-line summary, contact. Memorable. | 10 sec |

**Slide 4 is optional** — a strong pitch can end at slide 3 with the CTA baked in.

---

## Tier: lightning (6-8 slides, ~5 min)

**Context:** Meetup, demo day, sprint review, Cloud Native Rejekts, YC Demo Day, internal tech talk.

**Default structure — SCR (Situation-Complication-Resolution):**

| Slide | Purpose | Content | Duration |
|-------|---------|---------|----------|
| 1 | **Title** | Project name, one-liner, speaker name. | 10 sec |
| 2 | **Situation** | "We run X / We needed Y / The ecosystem has Z." Establish the world. | 30 sec |
| 3 | **Complication** | "But X is broken / slow / insecure / missing." The pain with evidence. | 45 sec |
| 4 | **Resolution** | What the project does. One sentence, one diagram. State the punchline early. | 30 sec |
| 5 | **How it works** | Architecture or mechanism — one simplified diagram (progressive disclosure). | 60 sec |
| 6 | **Demo/Screenshot** | The thing in action. Terminal output, UI screenshot, or benchmark result. | 45 sec |
| 7 | **Traction/Benchmarks** | Numbers that matter. Performance, adoption, reliability stats. | 30 sec |
| 8 | **CTA** | One URL (QR code). One sentence summary. "Questions?" | 10 sec |

**The one-idea rule:** a lightning talk with two ideas is two bad talks. Pick one message.

**Alternative framework — Lightning Formula (Codepipes):**
Slide 1: bold claim → Slides 2-3: pain → Slides 4-8: the trick → Slide 9: proof → Slide 10: one URL.

---

## Tier: talk (10-12 slides, ~20 min)

**Context:** Conference session, client presentation, workshop intro, KubeCon/GopherCon/RustConf talk.

**Default structure — 5-Box (What → Why → How → What If → What Next):**

| Slide | Purpose | 5-Box phase | Duration |
|-------|---------|-------------|----------|
| 1 | **Title** | — | 15 sec |
| 2 | **Hook / Problem** | What | 60 sec |
| 3 | **Context / Landscape** | Why — why this matters now | 90 sec |
| 4 | **Failed approaches** | Why — what doesn't work and why (builds trust) | 2 min |
| 5 | **Our approach** | How — the key insight | 2 min |
| 6 | **Architecture** | How — progressive disclosure diagram | 3 min |
| 7 | **Demo 1** | How — trivial "hello world" | 90 sec |
| 8 | **Demo 2** | How — realistic case | 2 min |
| 9 | **Results / Benchmarks** | What If — edge cases, failure modes, honest limitations | 2 min |
| 10 | **Lessons learned** | What If — what surprised us | 90 sec |
| 11 | **Roadmap** | What Next — where this goes | 60 sec |
| 12 | **CTA + Callback close** | What Next — reference the opening hook, close the loop | 30 sec |

**Alternative frameworks:**

- **PMR (Problem-Mechanism-Result):** for engineering-heavy talks. Precisely define the problem (metrics), show the mechanism, show the before/after.
- **Three-Act:** Act 1 (10%) = establish world + inciting incident. Act 2 (75%) = the journey, including a twist. Act 3 (15%) = resolution + lessons.
- **Pixar Pitch:** "Once upon a time, we had X. Every day, we Y. One day, Z. Because of that, W. Until finally, V." Adapted for tech.
- **Inverted Pyramid:** State the conclusion in slide 2, then layers of increasing detail. Best for internal talks where the audience wants the answer fast.

**The Rule of Three Demos:** in a 20-minute talk, do exactly three demos. Trivial → realistic → edge case. Never four.

---

## Tier detection — when the user doesn't specify

**Do not auto-detect.** Ask explicitly:

> "Which format? pitch (3-4 slides, ~2 min), lightning (6-8 slides, ~5 min), or talk (10-12 slides, ~20 min)?"

If the user gives a duration instead of a tier name:
- ≤ 3 min → pitch
- 4-7 min → lightning
- 8+ min → talk
