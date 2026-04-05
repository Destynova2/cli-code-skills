# Tooling Ladder -- Recommended Libraries by Language

> **When to read:** When recommending specific wizard implementations in audit recommendations.
>
> **Staleness:** Verified 2026-04-05. Library ecosystem moves fast. Recommendations (Huh, cliclack, questionary) are examples of good patterns, not prescriptions. If a library is no longer maintained, recommend its successor following the same design principles.

---

## Go -- The Charm Stack (State of the Art)

| Layer | Library | Purpose |
|-------|---------|---------|
| TUI framework | **Bubbletea** | Elm-architecture (Model-View-Update) TUI engine |
| Components | **Bubbles** | Reusable widgets (spinners, text inputs, viewports) |
| Styling | **Lip Gloss** | CSS-like terminal styling/layout |
| **Wizard forms** | **Huh v2** | Groups-as-pages, dynamic fields, accessible mode |
| Shell prompts | **Gum** | Composable prompts from bash (`gum input`, `gum choose`, `gum confirm`) |

**Huh v2** is the recommended choice for Go wizard implementations:
- Groups = pages -> multi-group forms = multi-step wizards automatically
- `TitleFunc` / `OptionsFunc` -> dynamic fields based on previous answers
- `.WithAccessible(true)` -> drops TUI for standard prompts (screen readers)
- Standalone or embedded in Bubbletea app

**Gum** is ideal for shell-based wizards (just/Makefile recipes, Bluefin/ujust pattern):
```bash
NAME=$(gum input --placeholder "Project name")
TYPE=$(gum choose "server" "client" "both")
gum confirm "Generate config for $NAME ($TYPE)?" && generate_config "$NAME" "$TYPE"
```

**Oz** (github.com/svyatov/oz): new config-driven wizard framework. YAML-defined flows built on Bubbletea. Early stage but architecturally interesting for declarative wizard definitions.

---

## Rust

| Library | Character | Best for | Watch out |
|---------|-----------|----------|-----------|
| **cliclack** | Modern, beautiful, Clack-inspired | New projects, best UX | Smaller ecosystem than dialoguer |
| **dialoguer** | Battle-tested, console-rs ecosystem | Production CLIs | Testability gap -- hard terminal dependency, tests hang |
| **inquire** | Feature-rich, complex inputs | Autocomplete, custom types | Needs feature flags, larger dependency tree |
| **promptly** | Minimal, FromStr-based | Simple scripts | Limited features |

**Recommendation:** **cliclack** for new wizard implementations. Themed output, input filtering, select/multiselect. If the project already uses dialoguer, don't switch -- but note the testability issue in audit.

---

## Python

| Library | Purpose | Best for |
|---------|---------|----------|
| **questionary** | Wizard-style interactive prompts | Setup flows (text, select, checkbox, path) |
| **Rich** | Output formatting + styled prompts | Visual quality, progress bars, tables |
| **Typer** + **Click** | Argument parsing | CLI structure, flags, help text |
| **Prompt Toolkit** | REPL/shell-style apps | Interactive shells, not wizard flows |

**Recommendation:** questionary for wizard prompts + Rich for output formatting + Typer for CLI structure.

---

## Node / TypeScript

| Library | Purpose | Notes |
|---------|---------|-------|
| **@clack/prompts** | Wizard flows | The original that cliclack (Rust) cloned. Gold standard |
| **inquirer.js** | Interactive prompts | Older, more plugins, less opinionated |
| **prompts** | Lightweight prompts | Minimal, good for simple flows |

**Recommendation:** @clack/prompts for new projects.

---

## Shell (any language)

| Tool | Pattern | Notes |
|------|---------|-------|
| **Gum** (Charm) | Composable shell prompts | Best option. Cross-platform, beautiful |
| **dialog** / **whiptail** | ncurses dialogs | Legacy, ugly, but universally available |
| **fzf** | Fuzzy finder as selector | Good for list selection, not full wizards |

**Bluefin/ujust pattern:** wizard = just recipes with Gum prompts. No framework, just composable bash steps.

---

## Framework comparison matrix

| Feature | Huh (Go) | cliclack (Rust) | questionary (Py) | @clack (Node) | Gum (shell) |
|---------|----------|-----------------|-------------------|---------------|-------------|
| Multi-step forms | Native groups | Manual | Manual | Manual | Compose commands |
| Dynamic fields | TitleFunc/OptionsFunc | Manual | Manual | Manual | Bash conditionals |
| Validation | Built-in | Built-in | Built-in | Built-in | Manual |
| Accessible mode | .WithAccessible(true) | No | No | No | --no-show-cursor |
| Testable | Bubbletea teatest | No (cliclack issue) | Mock stdin | Mock stdin | Bats/shunit2 |
| Non-interactive | Flags needed | Flags needed | Flags needed | Flags needed | Env vars / flags |
| Theming | Lip Gloss | Theme struct | Rich styling | Built-in | Gum flags |

---

## Reference implementations (study these)

| Project | Language | Pattern | Why it's good |
|---------|----------|---------|--------------|
| `step ca init` (Smallstep) | Go | Minimal seed wizard | 5 questions -> full PKI. Every question has a flag |
| OpenBSD installer | sh | Default-driven setup | Hit Enter through everything -> working system |
| `talosctl gen config` | Go | One-command config gen | cluster-name + endpoint -> full config. Idempotent |
| `rustup` | Rust | Setup + doctor lifecycle | `rustup-init` (setup) + `rustup check` (doctor) |
| `brew doctor` | Ruby | Modular doctor checks | OS-specific check plugins, independent execution |
| Caddy | Go | Live config API | REST CRUD on config tree, atomic rollback, no restart |
| Bluefin/ujust | shell/Gum | Recipe-based wizard | just + gum, no framework, composable steps |
