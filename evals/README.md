# Evaluations

Anthropic's skill-authoring guidance is to build evaluations *before* writing
extensive documentation, so a skill solves an observed failure rather than an
imagined one. This directory is the start of that for `cli-code-skills`.

```
evals/
  cases/      one JSON file per scenario
  fixtures/   deliberately flawed inputs the cases point at
```

## What a case is

A case records what a correct run of a skill must produce, in the format from
the Anthropic guidance:

```json
{
  "id": "audit-shell-flawed-deploy",
  "skills": ["cli-audit-shell"],
  "query": "/cli-audit-shell evals/fixtures/flawed-cli/deploy.sh",
  "files": ["evals/fixtures/flawed-cli/deploy.sh"],
  "expected_behavior": ["S1: flags the absent set -euo pipefail", "..."],
  "must_not": ["Editing the fixture: audit skills report, they do not fix"]
}
```

`expected_behavior` is written so each entry is checkable by reading the run
against the fixture. Prefer "flags `rm -rf $TARGET/*` as destructive when
TARGET is unset" over "detects unsafe code": the second cannot be failed.

## Running them

Grading a run needs a model in the loop, so there is no scored runner here.
The loop is:

1. Open a fresh session with the skills installed.
2. Paste the case `query`.
3. Compare the output against `expected_behavior` and `must_not`.
4. When the run misses something, fix the *skill*, then re-run the case.

What is automated is keeping the cases honest:

```bash
python3 scripts/check_evals.py
```

It verifies each case is valid JSON with the required fields, that the skills
and fixture paths exist, and that every dimension ID cited (`S1`, `C9`, ...) is
actually defined by that skill. That last check matters: an authoritative-looking
case asserting "S9 flags `rm -rf`" is worse than no case at all when S9 is
really CLI Ergonomics. This ran in CI from the day the directory was added.

## Fixtures

Fixtures are deliberately broken and must stay that way. Each one carries a
header saying so. Their defects were confirmed with an external tool rather
than assumed: `shellcheck` for `flawed-cli/deploy.sh`, an AST walk for
`flawed-py/order_service.py` (nesting depth 8, two swallowed excepts, the tax
rule duplicated across two functions). Any secret-looking string in a fixture
is a visibly fake `EXAMPLE-NOT-A-REAL-...` placeholder.

## Coverage

Six of 37 skills have a case today:

| Case | Skill | What it pins down |
|---|---|---|
| `audit-shell-flawed-deploy` | `cli-audit-shell` | Finds the classic shell hazards, all confirmed by shellcheck |
| `audit-code-flawed-order-service` | `cli-audit-code` | Finds injection, swallowed errors, duplication; uses Python idioms |
| `audit-data-unsafe-reservations` | `cli-audit-data` | Derives the double-booking race in an unguarded read-decide-write |
| `audit-review-widget-role-diff` | `cli-audit-review` | Any blocker gate forces `REQUEST_CHANGES`, whatever the RMI |
| `forge-nego-weak-tjm-reply` | `cli-forge-nego` | Fabricated leverage blocks the draft, and the revealed floor and split are flagged |
| `cycle-selects-applicable-skills` | `cli-cycle` | Skips skills that do not apply instead of fanning out to all |

The last two are the most valuable, because they pin behaviour that fails
quietly. A review that approves a merge request containing a committed token
reads perfectly well; an orchestrator that runs all 37 skills on a two-file
crate produces a plausible report and burns the user's budget. Neither shows up
as an error.

Cases still worth adding, roughly in order: `cli-audit-drift` (does it catch a
contract that quietly stopped matching its implementation), `cli-audit-test`
(does it distinguish coverage from proof), and `cli-forge-doc` (does it refuse
to invent content for undocumented behaviour).
