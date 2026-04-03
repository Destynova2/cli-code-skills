# Shell Anti-Patterns -- Named Patterns & Detection

> **When to read:** Step 5 (Detect anti-patterns). These are **semantic** patterns that shellcheck cannot find.

---

## Named Anti-Patterns

| # | Name | Severity | What it looks like | Why it's dangerous |
|---|------|----------|-------------------|-------------------|
| AP1 | **The Dead Fallback** | Critical | `set -euo pipefail` + `if ! command_a; then command_b; fi` where `command_a` failure exits before reaching `command_b` | Code looks like it handles errors but never executes the fallback path. False sense of safety |
| AP2 | **The Redundant Guard** | Major | Pre-checking what the tool already handles (`rpm -q pkg` before `dnf install`) | Adds complexity for zero benefit. dnf/yum return 0 if package is already installed |
| AP3 | **The Silent Swallower** | Critical | Systematic `2>/dev/null` or `|| true` on operational commands | Hides the exact error that would explain the failure. Debugging becomes guesswork |
| AP4 | **The Echo Logger** | Major | Custom `info()`, `warn()`, `error()` functions using `echo`/`printf` in scripts that run as system services | Output is lost after terminal closes. `logger(1)` integrates with journald/syslog for persistent, structured, queryable logs |
| AP5 | **The Namespace Squatter** | Major | Generic env var names (`DISK`, `USER`, `BASE_ROOT`) without project prefix | Collides with other tools on the same host. Imagine running MariaDB + MongoDB setup scripts — both define `DISK` |
| AP6 | **The Config Overwriter** | Critical | `cp my.conf ~/.config/tool/tool.conf` for shared config files | Another process (Ansible, Puppet, operator) may also manage this file. Your write destroys their config |
| AP7 | **The Sed Surgeon** | Major | Complex `sed` pipelines for structured data (YAML, JSON, XML) | Breaks on unexpected whitespace, comments, multi-line values. Use `yq`/`jq`/kustomize for structured formats |
| AP8 | **The Script Hydra** | Major | N separate scripts for related operations instead of one script with subcommands | Operator must know which script to run, in what order. Single entry point with `getopts` is self-documenting |
| AP9 | **The Heredoc Injector** | Critical | Shell variable interpolation inside heredocs/env blocks that get `source`d | If a variable contains `$(malicious_command)`, it executes when the env file is sourced. Use `printf %q` or structured formats |
| AP10 | **The Lazy Default** | Major | `${VAR:-dangerous_default}` for values that must be explicitly set by the operator | Silently uses the wrong disk, the wrong user, the wrong registry. Fail-fast with `${VAR:?message}` |
| AP11 | **The Local Hoarder** | Minor | `local name="$1"; local path="$2"; local mode="$3"` when each is used exactly once | Adds 3 lines of noise. Use `$1`, `$2`, `$3` directly when the function header comment documents them |
| AP12 | **The Ancient Wrapper** | Minor | `function cleanup { ... }` instead of `cleanup() { ... }`, backticks instead of `$()`, `[ ]` instead of `[[ ]]` | Outdated syntax. Google Shell Style Guide standardizes on modern bash idioms |
| AP13 | **The Platform Assumption** | Major | Using GNU-only features without portability check: `readlink -f` (no macOS), `sed -i` without `''` arg (BSD vs GNU), `date -d` (GNU-only), `realpath` (missing on older systems) | Script fails silently or with cryptic errors on non-Linux systems. Relevant when targeting mixed fleets (RHEL + macOS dev, containers + bare metal) |

---

## Detection Heuristics

### AP1 — Dead Fallback Detection

```
FIND: set -euo pipefail (or set -e) at script top
THEN: scan for patterns like:
  - if command -v X; then ... fi; if command -v Y; then ... fi
  - command_a || command_b (where command_a is not in an if/while condition)
VERIFY: is the second branch reachable under set -e?
  - If command_a is inside an if/while/|| condition: set -e is DISABLED there, branch IS reachable
  - If command_a is a standalone statement: set -e KILLS the script, branch is DEAD
```

### AP2 — Redundant Guard Detection

```
FIND: rpm -q <pkg> followed by dnf/yum install <pkg>
  OR: dpkg -s <pkg> followed by apt install <pkg>
  OR: command -v <tool> followed by install <tool>
CHECK: does the package manager already return 0 for "already installed"?
  - dnf/yum: YES, returns 0 and prints "Nothing to do"
  - apt: YES, returns 0
  - brew: YES
FLAG: the pre-check is redundant, remove it
```

### AP3 — Silent Swallower Detection

```
FIND: all instances of 2>/dev/null and || true
FOR EACH: is the silenced command operational (not just a probe)?
  - PROBES (OK to silence): command -v, type -t, hash, test/[
  - OPERATIONS (NOT OK): install, create, start, load, copy, move
FLAG: operational commands with silenced errors
```

### AP5 — Namespace Squatter Detection

```
FIND: top-level variable assignments (outside functions)
CHECK: is the name generic enough to collide?
  - Generic: DISK, USER, HOME, PORT, HOST, CONFIG, DATA, LOG, BACKUP, BASE
  - Namespaced: MARIADB_DISK, MONGODB_PORT, MYAPP_CONFIG
FLAG: generic names without project prefix
EXCEPTION: standard shell vars (PATH, HOME, SHELL, TERM, LANG)
```

### AP7 — Sed Surgeon Detection

```
FIND: sed commands operating on structured formats
HEURISTICS:
  - sed on files ending in .yaml, .yml, .json, .xml, .toml, .ini
  - sed patterns that match YAML keys: s/key:.*/key: value/
  - sed with more than 2 -e expressions on structured data
  - awk '{print $N}' on structured data (positional parsing)
RECOMMEND: yq for YAML, jq for JSON, xmlstarlet for XML
EXCEPTION: sed on plain text files, logs, or simple key=value configs is fine
```

### AP9 — Heredoc Injector Detection

```
FIND: heredocs or strings that:
  1. Contain variable references (${VAR} or $VAR)
  2. AND are written to files that will be sourced/executed later
PATTERNS:
  - echo "export FOO=\"${BAR}\"" >> file.sh
  - cat <<EOF > env.sh ... ${VAR} ... EOF  (double-quoted heredoc)
  - printf "export %s" "${VAR}" >> file  (if VAR could contain shell metacharacters)
CHECK: can an attacker control the interpolated variable?
FLAG: if YES, recommend printf %q or structured format (YAML parsed by yq)
NOTE: cat <<'EOF' (single-quoted heredoc) is SAFE — no interpolation occurs
```

### AP13 — Platform Assumption Detection

```
FIND: GNU-only commands and options:
  - readlink -f         → macOS readlink has no -f (use: python -c, perl, or $(cd ... && pwd))
  - sed -i 's/...'      → GNU sed. BSD sed requires: sed -i '' 's/...'
  - sed -i.bak 's/...'  → portable form (works on both)
  - date -d '...'       → GNU-only. BSD/macOS: date -j -f
  - date +%s%N          → %N not available on macOS
  - realpath             → missing on older macOS/busybox
  - grep -P              → PCRE not available everywhere. Use grep -E for extended regex
  - stat --format        → GNU syntax. BSD: stat -f
  - mktemp -d -t xxx    → template format differs between GNU and BSD mktemp
CHECK: does the script claim POSIX compatibility or target non-Linux systems?
  - If bash-only target (Google style): flag as INFO (awareness)
  - If multi-platform target: flag as MAJOR
EXCEPTION: scripts that explicitly require Linux (systemd units, LVM, RPM packaging)
```

---

## Severity Classification

| Severity | Meaning | Action |
|----------|---------|--------|
| Critical | Security risk, data loss, or silent failure in production | Fix immediately |
| Major | Maintainability risk, debugging difficulty, or operational fragility | Fix in next iteration |
| Minor | Style, clarity, or convention deviation | Fix when touching the file |
