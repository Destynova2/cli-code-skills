# Shell Quality Index (SQI) -- Scoring Framework

> **When to read:** Step 7 (Compute final score).

---

## SQI Formula

```
SQI = Sigma(wi x si) / Sigma(wi) x 10

Where:
  wi = weight of category i (adjusted for script type, see Step 2)
  si = score of category i (0.0 to 1.0)
```

## Scoring Guide

| Score | Meaning |
|-------|---------|
| 0.0 | Absent -- no evidence of this practice |
| 0.25 | Initial -- sporadic, inconsistent |
| 0.5 | Basic -- present but incomplete, significant gaps |
| 0.75 | Good -- consistent, minor issues remain |
| 1.0 | Excellent -- exemplary, could serve as Google Shell Style Guide reference |

## Category Weights (default, before type adjustment)

| # | Category | Weight | Rationale |
|---|----------|--------|-----------|
| S1 | Strict Mode Coherence | 12% | #1 source of "it works on my machine" bugs in bash |
| S2 | Error Surfaces & Return Values | 12% | Silent failures = 3am debugging sessions |
| S3 | Logging & Observability | 8% | Can you diagnose a failure from logs alone? |
| S4 | Stderr Hygiene | 5% | Foundation for piping and automation |
| S5 | Variable Discipline | 10% | Variable leaks, injection, masking = entire bug classes |
| S6 | Quoting & Expansion | 8% | #1 source of "works with my filenames" bugs |
| S7 | Control Flow & Structure | 8% | Maintainability for humans who didn't write it |
| S8 | Naming Conventions | 5% | Consistency, readability |
| S9 | CLI Ergonomics | 10% | Can an operator use this without reading the source? |
| S10 | Idempotency & Safety | 10% | Can you run this twice without breaking things? |
| S11 | Namespace & Env Hygiene | 7% | Multi-tool coexistence on same host |
| S12 | Security & Injection | 5% | eval, unquoted input, heredoc injection |

## Weight Adjustment by Script Type

| Script type | S1 | S2 | S3 | S9 | S10 | S12 |
|------------|------|------|------|------|------|------|
| Installer/bootstrap | x1 | x1 | x1 | x1.5 | x1.5 | x1.5 |
| Library (sourced) | x1 | x1 | x0.5 | x0 | x0.5 | x1 |
| Wrapper/glue (<50 lines) | x0.5 | x1 | x0.5 | x0.5 | x0.5 | x1 |
| Build/CI script | x1 | x1.5 | x1 | x0.5 | x1 | x1 |

After adjustment, re-normalize weights to sum to 100%.

## SQI Thresholds

```
8.0-10.0 -> Excellent  -- exemplary shell code, safe to hand to any operator
6.0-7.9  -> Good       -- solid with minor issues, production-worthy
4.0-5.9  -> Acceptable -- needs improvement, maintenance risk accumulating
2.0-3.9  -> Concerning -- significant quality issues, silent failures likely
0.0-1.9  -> Critical   -- major rewrite needed, dangerous in production
```

## Severity Classification

| Severity | Definition | Example |
|----------|-----------|---------|
| Blocker | Will fail silently in production | Heredoc injection, dead fallback on critical operation |
| Critical | Security vulnerability or data loss risk | `eval "$user_input"`, unquoted `rm $var` |
| Major | Operational fragility or debugging difficulty | Systematic `2>/dev/null`, missing getopts, unprefixed env vars |
| Minor | Style or convention deviation | Missing `readonly`, backtick syntax, `[ ]` instead of `[[ ]]` |
| Info | Suggestion, not a violation | "Consider consolidating these 3 scripts into subcommands" |

## Anti-Pattern Impact on Score

Each detected anti-pattern from `anti-patterns.md` reduces the relevant category score:

| Anti-Pattern | Category affected | Score reduction |
|-------------|------------------|----------------|
| AP1 Dead Fallback | S1 Strict Mode | -0.3 per instance |
| AP2 Redundant Guard | S2 Error Surfaces | -0.15 per instance |
| AP3 Silent Swallower | S2 Error Surfaces | -0.2 per instance |
| AP4 Echo Logger | S3 Logging | -0.25 if systematic |
| AP5 Namespace Squatter | S11 Namespace | -0.15 per generic name |
| AP6 Config Overwriter | S11 Namespace | -0.3 per overwrite |
| AP7 Sed Surgeon | (tooling flag) | Not scored, flagged in Tooling Recommendations |
| AP8 Script Hydra | S9 CLI Ergonomics | -0.3 if > 3 scripts |
| AP9 Heredoc Injector | S12 Security | -0.4 per instance |
| AP10 Lazy Default | S10 Idempotency | -0.2 per critical default |
| AP11 Local Hoarder | S5 Variables | -0.1 if systematic |
| AP12 Ancient Wrapper | S6 Quoting | -0.1 if systematic |
| AP13 Platform Assumption | S10 Idempotency | -0.15 per GNU-only pattern (if multi-platform target) |

## Comparative Benchmarks

| Project type | Typical SQI | Target |
|-------------|-------------|--------|
| Quick automation script | 3.0-5.0 | 5.0+ |
| CI/CD pipeline scripts | 4.0-6.0 | 6.0+ |
| Deployment/installer scripts | 5.0-7.0 | 7.0+ |
| Production operator tooling | 6.0-8.0 | 8.0+ |
| Infrastructure bootstrap (IaC) | 5.0-7.0 | 7.0+ |

## Sources

### Style Guides
- Google -- *Shell Style Guide* (primary authority for naming, quoting, structure)
- Wooledge -- *BashGuide*, *BashFAQ*, *BashPitfalls* (primary authority for bash semantics)

### Books
- Greg Wooledge -- *Bash Idioms* (O'Reilly, 2022)
- Carl Albing, JP Vossen -- *bash Cookbook* (O'Reilly, 2nd ed)

### Tools & Standards
- ShellCheck -- SC rule database (syntax/semantic linting)
- POSIX.1-2024 -- Shell Command Language specification
- Kfir Lavi -- *Defensive Bash Programming* (blog series)

### Online References
- Dylan Araps -- *Pure Bash Bible* (builtins over externals)
- mywiki.wooledge.org -- Authoritative bash reference
- explainshell.com -- Command decomposition tool
