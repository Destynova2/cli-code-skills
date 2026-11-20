# Anti-Patterns and Verdict

Run on the user's draft (`audit`) and on your own output before delivery.
Quote the offending line, name the pattern, give the fix.

## Blockers

Any blocker stops "ready to send", whatever the score:

| ID | Blocker | Example |
|---|---|---|
| B1 | Fabricated leverage | fake competing offer, fake deadline, invented scarcity, implied alternatives that do not exist, a walk-away the user has said they would not carry out |
| B2 | Invented fact | market figure, budget, clause, legal threshold, or approval not supplied by the user or cited |
| B3 | Threat or ultimatum | one the user has not explicitly decided on |
| B4 | Personal targeting | pressure on a private vulnerability of the counterpart |
| B5 | Private data leaked | the user's floor, BATNA, or another client's rate or name, shared as background, appears in outgoing text without permission |

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
| N8 | Closed or yes-seeking question | « Vous êtes d'accord ? », « Ça vous va ? » | Open « comment / qu'est-ce qui » question. In writing, never a no-oriented question: it reads sarcastic in French |
| N9 | Accusing "pourquoi" | « Pourquoi vous ne payez pas ? » | « Qu'est-ce qui bloque… ? » |
| N10 | Naked concession | « OK pour 30 jours de plus » with nothing in return | « Si…, alors… » |
| N11 | Reservation point revealed | « en dessous de X je ne peux pas » early; weak BATNA admitted | Remove |
| N12 | Several asks | unrelated requests in one message | One decision asked, first. A package or MESO is one ask |
| N13 | Buried ask | ask after more than one line of context | One framing line at most (a label or summary), then the ask |
| N14 | Single-issue haggling | only rate discussed | List the package |
| N15 | Accusing "vous" | « vous n'avez pas respecté… » | Label + dated facts |
| N16 | Hollow "je comprends" | « Je comprends tout à fait votre position » | Specific label |
| N17 | Premature take-it-or-leave-it | « à prendre ou à laisser » at first exchange | Label + question |
| N18 | Vague agreement | « on est d'accord » without price, scope, owner, date | Ask for operational confirmation |
| N19 | Mind-reading as fact | « il ment », « ils n'ont pas de budget » inferred from tone | Turn into a question to verify |
| N20 | Technique overload | three mirrors and four labels in one mail | Keep the one or two that serve the ask |
| N21 | Wrong register | Ackerman toward a manager; legal tone in a first payment reminder; Voss kit stacked in an interest-led message | Re-read the axes in SKILL.md |
| N22 | Trading what is owed | offering a fix of the user's own error, or a contractual deliverable, as a concession against something | Deliver or repair; trade only new requests |
| N23 | Anchor without a BATNA | ambitious figure while the user's alternative is unknown or weak | Ask for the BATNA first, or moderate the anchor |

## Verdict

No numeric score: weights would be decorative precision, and a polished
draft can still answer the wrong question. Give one of three verdicts, with
the reason in one line:

| Verdict | When |
|---|---|
| **Envoyer** | No blocker, no open question that changes the ask, defects only cosmetic |
| **Clarifier d'abord** | A fact the message depends on is missing: BATNA, unit, firm cap, what is owed, who decides. Name the one question to ask (the user, or the counterpart) |
| **Décision de l'utilisateur requise** | A blocker, or a move only the user can choose (walk-away, escalation, accepting below target) |

Then list at most three defects, most damaging first, each with the
offending line and the fix, followed by the rewrite. A blocker always
overrides **Envoyer**.
