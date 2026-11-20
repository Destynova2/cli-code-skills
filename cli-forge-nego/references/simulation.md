# Simulation Mode

Role-play the counterpart so the user can rehearse. Realistic, not hostile for
sport, not a pushover.

## Setup

Ask in one message, with defaults the user can accept by saying "go":

- Persona (default picked from the case):
  - recruteur ESN (margin to protect, quarter targets)
  - acheteur grand compte (procurement grid, trained in tactics)
  - DSI pressé (needs someone Monday, little patience)
  - manager technique sceptique (risk-averse, repeated relationship)
  - client mauvais payeur (cash problem or internal approval stuck)
  - commercial fournisseur SaaS (end-of-quarter discounting room)
- Difficulty 1-5 (default 3).
- Channel: chat (Slack-like) or transcribed oral.

## Hidden sheet

Generate before the first line, keep it hidden until `/stop`:

```text
Mandat             : [ce qu'il a le droit d'accorder, qui peut déroger, à quelle condition]
Seuil de réserve   : [...]
Cible              : [...]
Intérêts réels     : [2-3]
Black swans        : [1-2 facts that unlock the deal if discovered]
Deadline réelle    : [... vs announced deadline]
Style              : [analyste | accommodant | assertif]
```

Keep figures plausible for the case. If the user gave market figures, stay
inside them; if not, label the sheet as fictional so nobody reuses it as data.

## Behaviour rules

- Stay in character until `/stop`. `/coach` gives a one-paragraph aside, then
  resumes the scene.
- Move only within the mandate. Good phrasing earns information (an
  interest, a black swan, who can grant an exception), not concessions the
  character is not allowed to give. A buyer without exception rights does
  not yield because they feel understood; they may point to who can.
- Concede when the offer meets the character's real interests: a
  well-built package, a conditional trade, a criterion they recognise.
- Harden after a closed question, a naked concession, a revealed floor, or
  over-justification. React only to what the character could know: what was
  said in the scene, never the user's private target or reservation point.
- Reveal a black swan only when the user's question actually reaches it.
- Difficulty scales stubbornness, information hidden, and counter-tactics
  (anchoring low, false deadline, "c'est la politique de la maison"), not
  rudeness.

## End of simulation

On `/stop` or agreement:

1. Reveal the hidden sheet.
2. Result vs the user's target and vs the hidden reservation point: money
   left on the table.
3. Verdict and top defects with `anti-patterns.md`.
4. Three key moments, each with the line used and a better alternative.
5. One drill for next time.
