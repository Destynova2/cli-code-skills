# Defending a Project to Get a Decision

Use for a steering committee, a budget request, a scenario comparison, or
rehearsing a decision-maker's objections. Register: interest-led. No Ackerman,
no negative leverage: the audience will still be your management next quarter.

## Build the Ask

1. **Write the sentence the decision-maker could approve in the meeting.**
   State what they commit today: budget, team capacity, or a priority change.
   Keep three authorisations apart: to investigate, to experiment, to deploy.
2. **Set a verifiable baseline.** One observation tied to the business problem,
   with its source. Compare over the same horizon: status quo, its best
   realistic improvement, and the proposal. Keep only options that could
   actually be chosen.
3. **Show the whole bill.** Benefits, initial and recurring costs, lead time,
   execution risk, reversibility, and the load on the teams who will run it.
   Separate cash saved, time freed, and risk reduced. Count one person's time
   once. If the case rests on an estimate, name the variable that would flip
   the choice and its tipping value when it can be computed.
4. **Give each actor their real role.** Who funds? Who signs off technically?
   Who runs it after launch? Who settles a disagreement? Name the unknowns.
   Have the team concerned review the weakest assumption; do not presume their
   support.
5. **Open with the decision requested**, then the fact that justifies it, the
   best competing option, and the main risk. Close with the owner and the next
   decision point. Written note: one page by default. Short oral slot: lines
   that can be spoken in the time given.
6. **Pilot only if it removes an uncertainty that could flip the decision.**
   Bound its effort, scope, duration, baseline measure, test, continue/stop
   criteria, and owner. If the decision is already documented enough, ask for
   it directly.

## Read the Room

| Audience | Credible objection | Answer to build |
|---|---|---|
| CFO / finance | « Quelle dépense disparaît réellement cette année ? » | Point to the budget line and the horizon. If the gain is time, say who reuses it and for which priority. |
| CIO / executive | « Qu'est-ce qu'on retarde si on lance ça ? » | Name one capacity trade-off, the expected gain, and the option if deferred. |
| Operations / business | « Qui prend l'astreinte et qui revient en arrière ? » | Describe the change for the team, the owner to confirm, the real test, and rollback conditions. |
| Security / compliance | « Quelle preuve permettra de valider ? » | Name the known requirement, the test, and who may accept residual risk. |
| Sceptical sponsor | « La solution actuelle pourrait suffire. » | Give the best argument for it, and the result that would make you withdraw the recommendation. |

Acknowledge the valid part of the objection before answering. If a fact is
missing, propose who can verify it and when the decision can resume. If the
objection kills the option, change the proposal. A priority dispute is settled
by making the choice between two uses of a scarce resource explicit.

## Oral Example

Fictional case: an architect proposes a secrets-management pilot; leadership
hesitates between VMs and Kubernetes; no saving has been measured yet.

> « Je vous demande deux jours d'équipe pour comparer la restauration sur les
> VM et sur Kubernetes. On sait déployer les deux. Ce qui nous manque pour
> choisir, c'est qui peut rétablir le service d'astreinte, et en combien de
> temps. L'exploitation fait le même exercice sur les deux, avec le délai de
> reprise qu'elle aura validé. Si les VM tiennent ce délai avec moins de
> charge, je recommanderai de les garder. On réserve ces deux jours ? »

If the committee asks to approve the full rollout, surface the full resources
and run conditions. A pilot budget does not authorise deployment.

## Rehearsal

In `sim`, play first the decision-maker whose objection could legitimately
kill the project. In a committee, one voice at a time, each with an objection
specific to their function.
