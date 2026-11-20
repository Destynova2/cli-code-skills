# Anti-Patterns and Scoring

Run on the user's draft (`audit`) and on your own output before delivery.
Quote the offending line, name the pattern, give the fix.

## Blockers

Any blocker stops "ready to send", whatever the score:

| ID | Blocker | Example |
|---|---|---|
| B1 | Fabricated leverage | fake competing offer, fake deadline, invented scarcity, bluffed walk-away the user will not execute |
| B2 | Invented fact | market figure, budget, clause, legal threshold, or approval not supplied by the user or cited |
| B3 | Threat or ultimatum | one the user has not explicitly decided on |
| B4 | Personal targeting | pressure on a private vulnerability of the counterpart |

## Detector

| ID | Anti-pattern | French surface signals | Fix |
|---|---|---|---|
| N1 | Splitting the difference by reflex | « on coupe la poire en deux », « on se retrouve au milieu » offered early or as the user's own opener | Conditional counter. Not a defect when it is a deliberate closing move that meets `preparation.md` § When splitting the gap is fine |
| N2 | Self-negotiation | price lowered before any answer; « je peux faire un effort si besoin » | Hold the anchor, ask a question |
| N3 | Declared flexibility | « je suis flexible », « ouvert à toute proposition », « à discuter » | Anchor + criteria |
| N4 | Range at or below target | « entre 550 et 650 » with target 650 | Tandem range, low end = ambitious point offer |
| N5 | Unjustified number | a lone figure with no criterion, or decorative decimals nothing supports | Figure tied to 2-3 criteria; precise only when calculated |
| N6 | Over-justification | more than 3 sentences of argument, « parce que… parce que… » | 2-3 factual criteria |
| N7 | Softeners and apologies | « juste », « petit », « désolé de vous déranger », « si ce n'est pas possible, pas de souci » | Delete |
| N8 | Closed or yes-seeking question | « Vous êtes d'accord ? », « Ça vous va ? » | Calibrated or no-oriented question |
| N9 | Accusing "pourquoi" | « Pourquoi vous ne payez pas ? » | « Qu'est-ce qui bloque… ? » |
| N10 | Naked concession | « OK pour 30 jours de plus » with nothing in return | « Si…, alors… » |
| N11 | Reservation point revealed | « en dessous de X je ne peux pas » early; weak BATNA admitted | Remove |
| N12 | Several asks | unrelated requests in one message | One decision asked, first. A package or MESO is one ask |
| N13 | Buried ask | ask at the end under context | Move to the first two lines |
| N14 | Single-issue haggling | only rate discussed | List the package |
| N15 | Accusing "vous" | « vous n'avez pas respecté… » | Label + dated facts |
| N16 | Hollow "je comprends" | « Je comprends tout à fait votre position » | Specific label |
| N17 | Premature take-it-or-leave-it | « à prendre ou à laisser » at first exchange | Label + question |
| N18 | Vague agreement | « on est d'accord » without price, scope, owner, date | Ask for operational confirmation |
| N19 | Mind-reading as fact | « il ment », « ils n'ont pas de budget » inferred from tone | Turn into a question to verify |
| N20 | Technique overload | three mirrors and four labels in one mail | Keep the one or two that serve the ask |
| N21 | Wrong register | Ackerman steps toward a manager; legal tone in a first reminder | Switch register (SKILL.md Core Rule 3) |

## Scoring grid (/100)

| ID | Dimension | Points | Full marks when |
|---|---|---:|---|
| D1 | Preparation | 20 | Real BATNA, reservation point, justified target, hypotheses about them listed (`prep` and `debrief` only: not observable on a draft) |
| D2 | Anchor and numbers | 15 | Ambitious, backed by criteria, precise only when calculated |
| D3 | Discovery | 15 | Calibrated questions, labels, black-swan probes, summary |
| D4 | Concession discipline | 15 | Conditional, decreasing, no self-negotiation |
| D5 | Value creation | 10 | Multi-issue, MESO, non-monetary trades |
| D6 | Message clarity | 10 | One ask, first, no softeners, right length |
| D7 | Relationship and ethics | 10 | No lies, tone fits a repeated relationship, face-saving exit left open |
| D8 | Channel and culture | 5 | Oral vs written fit, French register, escalation choice |

For `reply` and `audit` on a short message, score only the dimensions the
message can show (D1 never is), then rescale: `score = points earned /
points available × 100`, rounded. List the skipped dimensions next to the
score.

Thresholds: ≥ 80 send; 60-79 fix the top two issues; < 60 go back to `prep`.
Any blocker overrides the score.
