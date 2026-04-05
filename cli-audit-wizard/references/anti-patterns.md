# Anti-Patterns -- Named Wizard UX Failures

> **When to read:** During Step 5 of the audit workflow. Flag each detected anti-pattern with severity and evidence.

---

## AP-1: Premature Interrogation

**Severity:** Critical

Asking users questions they cannot answer yet. Setup is the worst time for choices -- users have zero context about the system they're configuring.

**Detection:**
- Count questions in setup flow
- For each: can the user reasonably answer this before having used the system?
- Questions about internal implementation details (buffer sizes, thread counts, cache strategies)

**Examples:**
- "Choose your replication strategy: sync / async / semi-sync" (user hasn't seen the system yet)
- "Configure TLS cipher suites" (99% of users don't know or care)

**Fix:** Make it a default. Let users adjust post-setup via `--edit` or doctor mode.

> Reference: Stef Walter's "Installer Anti-Pattern" -- https://stef.thewalter.net/installer-anti-pattern.html

---

## AP-2: One-Shot Syndrome

**Severity:** Critical

Wizard runs once at install, then vanishes. No way to re-run, adjust, or check health. The wizard is a one-shot experience with no return path.

**Detection:**
- Run wizard with existing config file -- does it detect it?
- Look for `--doctor`, `--edit`, `--check` flags -- do they exist?
- Check: is there any post-setup health check?

**Examples:**
- Setup script that writes config and exits, no re-entry
- `init` command that refuses to run if config exists

**Fix:** Implement the full lifecycle state machine: setup (absent) -> doctor (present) -> edit -> migrate.

---

## AP-3: Hidden State

**Severity:** Critical

Wizard stores configuration in non-committable, non-inspectable locations. Database entries, binary blobs, environment variables scattered across shell profiles, Windows registry.

**Detection:**
- After wizard completes: is there a single config file on disk?
- Can you `cat` the config and understand it?
- Can you `git add` it?

**Examples:**
- Config stored only in SQLite database
- Wizard sets env vars in `.bashrc` but doesn't create a config file
- Config split across 5 different dotfiles with no single source of truth

**Fix:** One config file, one format (TOML/YAML), one location. Committable, diffable, readable.

---

## AP-4: Flag-Less Prompts

**Severity:** Major

Interactive prompts with no corresponding CLI flag. The wizard cannot run in CI, cannot be scripted, cannot be automated via MCP.

**Detection:**
- List all interactive prompts in the wizard
- For each: is there a `--flag` that provides the same value non-interactively?
- Try running the wizard in a pipe (`echo "" | wizard`) -- does it hang?

**Examples:**
- `wizard setup` hangs in CI because it waits for interactive input
- No `--yes` or `--accept-defaults` flag for non-interactive mode

**Fix:** Every prompt gets a flag. Add `--yes` / `--accept-defaults` for full non-interactive mode.

---

## AP-5: Derivable Questions

**Severity:** Major

Asking questions whose answers can be computed from previous answers or environment detection.

**Detection:**
- For each question: is the answer deterministic given prior answers?
- Could the wizard detect this from the environment (OS, installed tools, running services, network config)?

**Examples:**
- Asking for "database host" when a PostgreSQL socket is detected at the standard path
- Asking for "TLS certificate path" when Let's Encrypt auto-provision is available
- Asking for "listen port" when the standard port is available

**Fix:** Derive and use as default. Only show the derived value in recap, let user override there.

---

## AP-6: Null Defaults

**Severity:** Major

Config keys with empty string, `null`, `""`, `0`, or placeholder defaults. Forces the user to make a decision about every field.

**Detection:**
- Parse config template/schema
- Find keys where default is empty, null, zero, or contains "CHANGEME", "TODO", "your-value-here"

**Examples:**
- `api_key = ""` (should not be in config template -- prompt only if needed)
- `log_level = ""` (should default to `info`)
- `port = 0` (should default to standard port for the service)

**Fix:** Every key has an opinionated, documented default. If a value can't have a sensible default (like API keys), don't include it in the template -- prompt for it in setup flow only.

---

## AP-7: Silent Apply

**Severity:** Major

Wizard writes config and applies changes without showing the user what changed. No recap, no diff, no confirmation.

**Detection:**
- Run wizard: does it show a summary before writing?
- Is there a `--dry-run` flag?
- Can the user abort after seeing the summary?

**Examples:**
- Wizard writes Caddyfile and restarts Caddy without showing the new config
- Setup modifies firewall rules silently

**Fix:** Always recap before apply. Implement `--dry-run`. Show a diff when re-running with existing config.

---

## AP-8: Surface Divergence

**Severity:** Major

Multiple configuration surfaces (CLI, web, MCP) that use different validation logic, different config formats, or write to different locations.

**Detection:**
- Configure via CLI, then read via web API -- same values?
- Configure via MCP, then `cat` the config file -- consistent?
- Change a value via web, re-run CLI wizard -- does it see the change?

**Examples:**
- CLI writes TOML, web API writes JSON to a different file
- CLI validates port range, MCP tool doesn't
- Web UI has config options that CLI doesn't expose

**Fix:** One engine, one config file, one validation layer. All surfaces are thin wrappers.

---

## AP-9: Deep IVR Nesting

**Severity:** Minor

Multi-tier question cascades where users navigate through layers of sub-menus to answer one basic question. Named after the telephone IVR systems everyone hates.

**Detection:**
- Count nesting depth of wizard flow (question -> sub-question -> sub-sub-question)
- Any wizard path longer than 3 levels deep?

**Examples:**
- "Configure backends" -> "Choose type" -> "Configure HTTP" -> "Advanced options" -> "Timeout settings"

**Fix:** Flatten. Use smart defaults for deep options. Only surface top-level choices, everything else is `--advanced` or post-setup `--edit`.

---

## AP-10: Dishonest Progress

**Severity:** Minor

Progress indicators that don't reflect actual state. Spinners with no status, progress bars that lie, "almost done" that lasts 5 minutes.

**Detection:**
- Does the wizard show progress during long operations?
- Does the progress indicator reflect actual work done?
- Is there a way to know what the wizard is currently doing?

**Examples:**
- Spinner with no message during 30-second TLS cert generation
- "Step 3 of 3" but step 3 has 12 sub-steps

**Fix:** Show what's happening. Use step counters or named phases. If an operation takes > 2 seconds, show a status message.

---

## Summary

| # | Anti-Pattern | Severity | Core Violation |
|---|-------------|----------|---------------|
| AP-1 | Premature Interrogation | Critical | Law 1 (ask once) |
| AP-2 | One-Shot Syndrome | Critical | Lifecycle (no doctor) |
| AP-3 | Hidden State | Critical | Law 4 (config-as-code) |
| AP-4 | Flag-Less Prompts | Major | Scriptability |
| AP-5 | Derivable Questions | Major | Law 1 (ask once) |
| AP-6 | Null Defaults | Major | Law 2 (defaults are decisions) |
| AP-7 | Silent Apply | Major | Law 3 (recap before apply) |
| AP-8 | Surface Divergence | Major | Multi-surface consistency |
| AP-9 | Deep IVR Nesting | Minor | UX flow |
| AP-10 | Dishonest Progress | Minor | UX honesty |
