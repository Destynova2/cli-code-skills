# Claude Code Skills: cleancode & doc-audit

Reusable [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skills for code quality auditing.

## Skills

### `/cleancode [file-or-directory]`

Audit code against Clean Code principles (Robert C. Martin, John Ousterhout, Martin Fowler, SonarSource, Casey Muratori).

Scores 10 categories: Naming, Functions, Comments, Structure, DRY, Error Handling, Rust Idioms, Tests, Immutability, Cognitive Load.

### `/doc-audit [file-or-directory]`

Audit documentation quality against industry standards (RFC 505, RFC 1574, Microsoft M-DOC, Diataxis, Clean Code, Linux Kernel).

Scores 6 categories: Doc Coverage, Summary Lines, Standard Sections, Inline Comments, Accuracy & Freshness, Cross-references.

Works with any language (Rust, TypeScript, Python, Go, Java, Kotlin).

## Installation

Copy skill directories into your Claude Code skills folder:

```bash
# Clone
git clone https://github.com/Destynova2/claude-code-skills.git

# Copy to your Claude Code skills directory
cp -r claude-code-skills/cleancode ~/.claude/skills/
cp -r claude-code-skills/doc-audit ~/.claude/skills/
```

Then use `/cleancode` or `/doc-audit` in any Claude Code session.

## Output

Both skills produce structured markdown reports with:
- Overall score out of 10
- Per-category scores table
- Critical violations with `file:line` references
- Flagged issues grouped by category
- Good practices found
- Recommended next steps

## License

MIT
