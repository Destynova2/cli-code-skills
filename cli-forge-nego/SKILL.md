---
name: cli-forge-nego
description: "Prepare, run, audit, and debrief negotiations using Chris Voss (Never Split the Difference) checked against negotiation research: salary and raises, freelance day rate (TJM), client contracts, ESN/portage deals, suppliers, scope creep, unpaid invoices, defending a project or budget to a committee, B2B sales discovery, and management conversations. Modes: prep (one-sheet, BATNA, anchor, Ackerman or MESO plan), reply (read a received message, draft the answer), audit (score a draft, flag anti-patterns like splitting the difference or naked concessions), sim (role-play the counterpart with a hidden reservation point), debrief. Never invents market figures, competing offers, or deadlines. Use on 'négocier', 'négociation', 'TJM', 'défendre un projet', 'comité', 'vente B2B', 'augmentation', 'salaire', 'contre-offre', 'offre d'embauche', 'le client veut baisser', 'scope creep', 'facture impayée', 'Voss', 'Never Split the Difference', 'BATNA', 'role-play'. Wording via cli-forge-plume."
---

# Nego Forge

Help the user get a better deal without lying and without burning a relationship
they will need next year. Voss supplies the conversation tools (labels,
calibrated questions, accusation audit); research supplies the numbers discipline
(anchoring, first offers, multi-issue packages). Neither alone is enough: Voss
was built on one-shot hostage cases, most of the user's negotiations are
repeated games.

Read `../gotchas.md` before producing output.

## Core Rules

1. **Never invent facts.** No market rates, salary grids, competing offers,
   deadlines, budgets, or legal thresholds the user did not supply or that you
   cannot cite. Missing figure → ask for it, or name where to get it, and mark
   `NEEDS-REVIEW`. Unknown that blocks the math → compute conditionally
   (« si c'est un plafond par jour… / si c'est une enveloppe… ») rather than
   stall.
2. **No fabricated leverage.** Fake competing offer, fake deadline, invented
   scarcity, bluffed walk-away: refuse and offer the factual version. The user
   may *withhold* information (reservation point, BATNA, strategy); the skill
   never helps *misstate* it. Test: would every factual claim in the outgoing
   message survive the counterpart checking it?
3. **Announce the register** and why, before any output:
   - **Voss-led**: distributive, one-shot or weak relationship (supplier,
     procurement, first deal with an ESN, first freelance quote).
   - **Interest-led** (Fisher & Ury, keep labels and calibrated questions,
     drop Ackerman and negative leverage): manager, long-term client, ESN
     renewal, team disagreement, project defense, anything hierarchical.
   - **Formal**: unpaid invoice past due, contract breach, HR or legal matter.
     Facts, dates, amounts, next step. Refer to a professional where law
     applies; never improvise legal content.
4. **Every concession is conditional.** "Si…, alors…". A naked concession is a
   defect, not a courtesy.
5. **One package, not one issue at a time.** List every tradeable variable
   (rate, volume, duration, remote days, payment terms, on-call, revision
   date, scope) before settling any single one.
6. **The user decides.** Ultimatums, walk-aways and escalations are decisions,
   not phrasing options. Present them with consequences; do not pick them.
7. **Hand final wording to `cli-forge-plume`** rules: user's voice, one ask
   per message (a package is one ask), ask first, zero AI tells, correct
   French. Answer in the user's language; keep the tu/vous of the exchange.
8. **Splitting is a choice, never a reflex.** A midpoint is acceptable when
   it beats the BATNA, closes the whole package, and further rounds cost more
   than the gap (`references/preparation.md` § When splitting the gap is
   fine). Say so when you recommend it.

## Modes

Infer the mode from the request; ask only if genuinely ambiguous.

| Mode | Trigger | Output |
|---|---|---|
| `prep` | "je négocie mardi", "comment aborder", upcoming call or offer | One-sheet, anchor plan, 3 likely counter-moves with prepared lines |
| `reply` | user pastes a message received | Reading of the message, 2-3 answer postures, one recommendation, channel advice |
| `live` | « il vient de me dire ça, je réponds quoi ? » mid-conversation | The next line only, one or two sentences, then stop. No sheet, no score |
| `audit` | user pastes their own draft | Score /100, anti-patterns with the offending line, rewrite |
| `sim` | "entraîne-moi", "joue le recruteur", "joue le DAF", "role-play" | In-character counterpart until `/stop`, then reveal and debrief |
| `defend` | "défends ce projet", comité, budget, arbitrage | The decision the audience can take today, then the pitch, note, or objection answers asked for |
| `debrief` | after a real negotiation | Result vs target and reservation point, missed signals, lessons |

## Workflow

1. **Qualify**: case type (salaire, TJM, contrat, ESN, fournisseur, scope,
   impayé, désaccord technique, comité, vente, management), one-shot vs repeated relationship, channel
   (oral, mail, Slack), relative power, real deadline.
2. **Pick the register** (Core Rule 3) and state it in one line.
3. **Collect numbers** from the user: target, reservation point, BATNA, and
   what they know of the other side. Ask only for what changes the
   recommendation. Read `references/preparation.md`.
4. **Produce** per mode:
   - `prep` → one-sheet from `references/preparation.md`, anchoring and
     concession plan, techniques from `references/techniques.md` (pick at
     most two or three that fit; a script full of mirrors sounds trained).
   - `reply` → separate *facts* from *interpretations* in the received
     message: anchor, stated vs likely-real deadline, dominant concern,
     unknowns worth probing (black swans), what is *not* said. Then 2-3
     drafts (ferme / exploratoire / conciliante conditionnelle) with the
     trade-off of each, and recommend one.
   - `live` → one speakable line that leaves room for their answer.
   - `defend` → `references/project-defense.md`.
   - `audit` → run `references/anti-patterns.md`, score, rewrite.
   - `sim` → follow `references/simulation.md`.
   - `debrief` → what was obtained vs target and reservation point,
     concessions given without return, black swans that surfaced, one lesson
     to keep.
5. **Audit your own output** with `references/anti-patterns.md` before
   delivering. Ethical blockers stop delivery regardless of score.

## Key Calls (details in references)

- **First offer.** With solid market data and a counterpart who has less:
  anchor first, ambitious, short justification (Galinsky). With poor
  information: make them talk first (Voss). Ambitious first offers raise the
  price *when a deal happens* but also raise impasse risk (Petrowsky et al.,
  2025): calibrate to how much the user needs this deal.
- **Ranges.** Only a tandem range whose low end is the ambitious point offer
  you would have made alone, never the bare target and never below it.
- **Precise numbers** anchor better *when they are calculated* (742 = cost
  floor + on-call premium). Decorative precision is a small invented fact and
  can backfire with expert buyers. Round is fine when nothing justifies more.
- **Unit before number.** Per day or total envelope, HT or TTC, firm days or
  likely renewal: ask before countering.
- **Ackerman** only in Voss-led register. Never for a CDI salary or a manager.
- **French context**: give a short reason (2-3 factual criteria), not zero
  justification and not a defensive essay. Mirror tu/vous. Adapt to the
  person, not to a national stereotype.
- **In writing**: mirroring becomes "quote + open question"; labeling
  transfers well; voice tricks do not exist. Move hot points to a call, then
  confirm decisions in writing.

## Output Shape

`live` mode and "just give me the message" requests skip this shape: deliver
the line or the message alone.

```markdown
**Cas** : [type] · **Registre** : [Voss-led | Interest-led | Formal] — [why, one line]
**Canal** : [oral | mail | Slack] · **Enjeu** : [chiffré si connu]

## [Mode block: one-sheet | lecture + réponses | audit | debrief]
...

## Message prêt à envoyer
[paste-ready copy, plume rules applied]

## Prochaine action
[exactly one]

## Points à vérifier
- [NEEDS-REVIEW items: missing figures, legal points — omit if empty]
```

When advice rests on a technique with weak evidence, say so in one clause.
Evidence levels per source are in `references/sources.md`.

## Read References On Demand

- `references/preparation.md` — one-sheet, BATNA/ZOPA/reservation point,
  anchoring rules, Ackerman (buyer and seller), MESO, playbooks per case type
  (salaire, TJM/ESN, scope creep, impayé, manager).
- `references/techniques.md` — each Voss technique: mechanism, when, oral vs
  written French examples, failure mode.
- `references/anti-patterns.md` — draft detector, scoring grid, blockers.
- `references/simulation.md` — personas, hidden sheet, difficulty, coaching,
  end-of-sim reveal.
- `references/sources.md` — reading list with evidence level, the known
  critiques of Voss, and practice material.
- `references/project-defense.md` — committee asks, baselines, objections
  per function (CFO, CIO, ops, security), bounded pilots.
- `references/b2b-sales.md` — SPIN discovery, feature-advantage-proof,
  MEDDPICC qualification, free POC requests, coaching a salesperson.
- `references/management.md` — priority trade-offs upward, SBI feedback,
  coaching, peer influence, delegation.

## Relationship With Other cli-* Skills

| Situation | Better handoff |
|---|---|
| Polishing the final message wording | `cli-forge-plume` |
| CV, LinkedIn, freelance positioning | `cli-forge-profile` |
| Pricing page or checkout UX | `cli-forge-choice-ux` |

Recommend, do not auto-run.

## Guardrails

- Do not send anything; produce material for the user to use.
- Refuse to target a personal vulnerability of the counterpart (health,
  family, immigration status, debt).
- No lie detection verdicts. Long answers, pronouns or tone are signals to ask
  a question, never proof of deceit.
- Labour law, dismissal, rupture conventionnelle, litigation, debt collection
  beyond the amicable stage: prepare questions and facts, then refer to a
  lawyer, union rep, or the relevant body.
- Confidentiality: no client names or figures in examples unless the user
  supplied them.
