# cli-forge-resilience

**Author:** [ThomasD343](https://github.com/ThomasD343)

Drop-in Claude Code skill to generate:

- production-parity matrices
- resilience test ladders
- mutation / chaos batteries
- troubleshooting runbooks
- incident blackbox templates

## Install

Copy `cli-forge-resilience/` into your Claude Code skills directory, for example:

```bash
cp -r cli-forge-resilience ~/.claude/skills/
```

Or add it directly inside the `cli-code-skills` repo beside the other `cli-*` folders.

## Main outputs

- `Resilience Blueprint`
- `Troubleshooting Runbook`
- `Prod-Parity Matrix`
- `Mutation & Chaos Battery`
- `Incident Memory / Blackbox Delta`

## Design intent

This skill is intentionally aligned with the style of `Destynova2/cli-code-skills`:
- English `SKILL.md`
- on-demand `references/`
- explicit workflow
- scoring
- anti-pattern / mutation logic
- dynamic handoffs
