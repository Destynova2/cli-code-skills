# Shell Audit Categories -- Detailed Checks

> **When to read:** Step 4 (Score all dimensions). Contains the full checklists per category.
> **Primary authority:** Google Shell Style Guide. Deviations are noted.

---

## S1: STRICT MODE COHERENCE (12%)

**Source:** Google Shell Style Guide (use `set` for shell options), Wooledge BashFAQ

The most weighted category because `set -euo pipefail` changes bash semantics fundamentally. Code that ignores this creates **dead branches** — code that looks functional but can never execute.

Check for:

- `set -euo pipefail` present at script top? (or equivalent individual `set -e`, `set -u`, `set -o pipefail`)
- **Dead fallbacks under `set -e`**: patterns like `command -v foo || command -v bar` where the `||` branch is unreachable because `set -e` already exits on the first failure
  ```bash
  # ANTI-PATTERN: set -e kills the script before reaching the fallback
  set -euo pipefail
  if command -v dnf >/dev/null 2>&1; then
    dnf install "$@"
  fi
  if command -v yum >/dev/null 2>&1; then  # DEAD: if dnf ran and failed, script already exited
    yum install "$@"
  fi
  ```
  **Fix:** Use explicit `||` chains or a function that returns status without triggering `set -e`:
  ```bash
  install_pkg() {
    dnf install -y "$@" 2>/dev/null || yum install -y "$@"
  }
  ```
- **`(( i++ ))` trap**: with `set -e`, `(( i++ ))` exits if `i` was 0 (expression evaluates to 0 = false = exit). Use `(( i += 1 ))` instead
- **Subshell exit masking**: `local var=$(failing_command)` — `local` swallows the exit code. Separate declaration from assignment:
  ```bash
  local my_var
  my_var="$(my_func)"
  ```
- `set -e` in functions that use `||` or `&&` guards — understand that `set -e` is disabled inside the condition of `if`, `while`, `||`, `&&`
- Scripts callable as `bash script_name` (shebang + `set` for options, not flags on shebang)

## S2: ERROR SURFACES & RETURN VALUES (12%)

**Source:** Google Shell Style Guide (Checking Return Values), SonarQube Reliability

Check for:

- **Blanket `|| true`**: Silences ALL errors, not just expected ones. Replace with specific handling
- **Blanket `2>/dev/null`**: Hides diagnostic info. Only acceptable on specific commands where stderr is expected noise (e.g., `command -v`)
- **Unchecked return values**: Commands whose failure is silently ignored
  ```bash
  # BAD: if mv fails, script continues with inconsistent state
  mv "${files[@]}" "${dest}/"

  # GOOD: check return value
  if ! mv "${files[@]}" "${dest}/"; then
    err "Unable to move files to ${dest}"
    exit 1
  fi
  ```
- **PIPESTATUS unused**: In pipelines, only the last command's exit code is checked by default
- **Missing `trap` cleanup**: Scripts that create temp files/dirs without `trap cleanup EXIT`
- **`|| true` on critical operations**: Package installs, file operations, service starts
- **Redundant existence checks**: `rpm -q pkg || install pkg` when the package manager already handles "already installed" idempotently (yum/dnf return 0 if already installed)
  ```bash
  # REDUNDANT: dnf already handles installed packages
  if ! rpm -q "${pkg}" >/dev/null 2>&1; then
    dnf install -y "${pkg}"
  fi

  # SIMPLER: let dnf handle it
  dnf install -y "${pkg}"
  ```

## S3: LOGGING & OBSERVABILITY (8%)

**Source:** ops best practices, 12-factor app logging

Check for:

- **Custom echo-based logging vs `logger(1)`**: For scripts running as system services (systemd, cron), `logger -t scriptname` integrates with journald/syslog. Custom `echo`/`printf` logging is acceptable for interactive scripts only
  ```bash
  # INTERACTIVE SCRIPT: echo is fine
  info() { printf '[INFO] %s\n' "$*"; }

  # SYSTEM SERVICE SCRIPT: use logger
  info() { logger -t "${SCRIPT_NAME}" -p user.info "$*"; }
  err()  { logger -t "${SCRIPT_NAME}" -p user.err "$*"; }
  ```
- **No timestamps**: Interactive output without timestamps makes debugging impossible
- **No severity levels**: All output at same level (no distinction info/warn/error)
- **Color codes without TTY/NO_COLOR gating**: Scripts using ANSI escapes (`\033[0;31m`) without checking context. Must respect the NO_COLOR standard (https://no-color.org/) and terminal capability
  ```bash
  # BAD: color codes always emitted, even in cron/CI/pipe
  info() { echo -e "\033[0;32m[OK]\033[0m $*"; }

  # GOOD: gate on TTY, NO_COLOR, and TERM
  if [ -t 1 ] && [ -z "${NO_COLOR:-}" ] && [ "${TERM:-}" != "dumb" ]; then
    GREEN='\033[0;32m'; NC='\033[0m'
  else
    GREEN=''; NC=''
  fi
  info() { printf '%b[OK]%b %s\n' "${GREEN}" "${NC}" "$*"; }
  ```
- **Missing `--verbose`/`--quiet` flags**: Scripts with no way to control output level

## S4: STDERR HYGIENE (5%)

**Source:** Google Shell Style Guide (STDOUT vs STDERR)

> "All error messages should go to STDERR."

Check for:

- **Error messages to stdout**: `echo "Error: ..."` instead of `echo "Error: ..." >&2`
- **Systematic stderr suppression**: `2>/dev/null` on commands where errors are diagnostic
  ```bash
  # BAD: hides WHY augenrules failed
  augenrules --load >/dev/null 2>&1 || info "Audit rules left unchanged"

  # GOOD: let stderr flow, handle specific error
  if ! augenrules --load 2>&1; then
    warn "augenrules failed — check audit daemon status"
  fi
  ```
- **Mixed stdout/stderr**: Script output that mixes operational data with diagnostics, making piping unreliable
- Google recommends a helper function:
  ```bash
  err() {
    echo "[$(date +'%Y-%m-%dT%H:%M:%S%z')]: $*" >&2
  }
  ```

## S5: VARIABLE DISCIPLINE (10%)

**Source:** Google Shell Style Guide (Variable expansion, Use Local Variables)

Check for:

- **Missing `local`**: Variables in functions that leak to global scope
  ```bash
  # BAD: device, mount leaks globally
  ensure_lv() {
    device="$1"
    mount="$2"
  }

  # GOOD
  ensure_lv() {
    local device="$1"
    local mount="$2"
  }
  ```
- **Excessive `local`**: Creating named locals for positional parameters used only once — `$1` is clearer
  ```bash
  # OVERKILL for single-use:
  my_func() {
    local input="$1"
    echo "${input}"
  }

  # CLEARER:
  my_func() {
    echo "$1"
  }
  ```
  Note: If the parameter is used more than once or the name adds clarity, `local` is correct.
- **Shell injection in heredocs/env blocks**: Variables interpolated into shell source that gets `source`d or `eval`d
  ```bash
  # DANGEROUS: if RUNTIME_ROOT contains $(malicious_command), it executes on source
  env_block="export MARIADB_DIR=\"${RUNTIME_ROOT}\""
  echo "${env_block}" >> ~/.bashrc

  # SAFER: use a non-interpolated format or validate inputs
  printf 'export MARIADB_DIR=%q\n' "${RUNTIME_ROOT}" >> ~/.bashrc
  ```
- **`declare` vs `local`**: Google recommends `readonly` or `export` over equivalent `declare` commands for clarity
- **`local` + command substitution on same line**: `local var="$(cmd)"` — the `local` builtin masks `cmd`'s exit code. Separate:
  ```bash
  local var
  var="$(cmd)"
  ```

## S6: QUOTING & EXPANSION (8%)

**Source:** Google Shell Style Guide (Quoting, Variable expansion)

Check for:

- **Unquoted variable expansions**: `$var` instead of `"${var}"` (except in `$(( ))` arithmetic)
- **`$*` instead of `"$@"`**: Loses argument boundaries
- **Unquoted command substitutions**: `var=$(cmd)` instead of `var="$(cmd)"`
- **Missing braces on multi-char variables**: `$var` instead of `${var}` (except single-char specials like `$?`, `$#`, `$!`)
- **Backticks instead of `$(...)`**: `` `cmd` `` instead of `$(cmd)`
- **String-based argument lists instead of arrays**:
  ```bash
  # BAD: breaks on arguments with spaces
  flags='--foo --bar=baz'
  mybinary ${flags}

  # GOOD: preserves argument boundaries
  declare -a flags=(--foo --bar='baz')
  mybinary "${flags[@]}"
  ```
- **Unquoted glob patterns**: `rm *` instead of `rm ./*` (filenames starting with `-`)
- **Pattern matching in `[[ ]]`**: RHS must NOT be quoted for pattern matching, MUST be quoted for literal comparison

## S7: CONTROL FLOW & STRUCTURE (8%)

**Source:** Google Shell Style Guide (Function Location, main, Control Flow)

Check for:

- **No `main()` function**: Scripts > 50 lines without a `main` function. Last line should be `main "$@"`
- **Executable code between functions**: Functions should be grouped at top, after constants/includes
- **`; then` / `; do` placement**: Must be on same line as `if`/`for`/`while` (Google style)
  ```bash
  # GOOD
  if [[ "${var}" == "val" ]]; then

  # BAD
  if [[ "${var}" == "val" ]]
  then
  ```
- **Deep nesting**: > 3 levels of `if/for/while` — extract to function
- **Missing `in "$@"`**: `for arg; do` instead of `for arg in "$@"; do` (less clear)
- **Case statement formatting**: Indent alternatives by 2 spaces, `;;` on its own line for multi-line actions
- **Script length**: > 100 lines should be evaluated for rewrite in a more structured language (Google guideline)

## S8: NAMING CONVENTIONS (5%)

**Source:** Google Shell Style Guide (Naming Conventions)

Check for:

- **Function names**: Must be `lower_snake_case`. Library functions use `package::function_name`
- **Variable names**: `lower_snake_case` for locals, `UPPER_SNAKE_CASE` for constants and exports
- **Constants not `readonly`**: Values set once and never modified should use `readonly` or `declare -r`
  ```bash
  readonly SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
  ```
- **Source filenames**: `lower_snake_case` with `.sh` extension for libraries, no extension for executables in PATH
- **Inconsistent naming**: `mixedCase`, `kebab-case` (filenames only, not variables), `camelCase`
- **Non-descriptive loop variables**: `for i in "${items[@]}"` instead of `for item in "${items[@]}"`

## S9: CLI ERGONOMICS (10%)

**Source:** Google Shell Style Guide (When to use Shell), ops best practices

Check for:

- **No `--help` / usage()**: Scripts without help text
- **No `getopts`/`getopt`**: Scripts with > 2 arguments that use positional parameters instead of named options
  ```bash
  # BAD: caller must remember order
  ./setup.sh svc-mariadb /dev/sdc registry.example.com

  # GOOD: self-documenting
  ./setup.sh --user svc-mariadb --disk /dev/sdc --registry registry.example.com
  ```
- **Multiple entry point scripts**: N separate scripts (`hardened_podman_install.sh`, `init-storage.sh`, `install-host-compat.sh`) that should be subcommands of one script
  ```bash
  # INSTEAD OF:
  ./hardened_podman_install.sh
  ./init-storage.sh
  ./install-host-compat.sh

  # CONSIDER:
  ./setup.sh podman-install
  ./setup.sh init-storage
  ./setup.sh host-compat
  ```
- **No input validation**: Missing argument count checks, missing `[[ -b "${DISK}" ]]` for block devices
- **Hardcoded values without override**: Values that should be configurable but are buried in the script
- **No `--version` flag**: Professional scripts should expose version metadata
- **No structured output option**: Scripts that produce operational output should consider `--json`/`--plain` for programmatic consumption (see clig.dev). Not mandatory for all scripts, but valuable for tools that feed into pipelines or monitoring
- **No `--dry-run` flag**: Destructive scripts (storage init, service install) should support a dry-run mode to show what would happen without executing

## S10: IDEMPOTENCY & SAFETY (10%)

**Source:** infrastructure-as-code principles

Check for:

- **Non-idempotent operations**: `useradd` without `id -u` check, `mkdir` without `-p`, `ln` without `-f`
- **Missing `readonly`**: Constants that could be accidentally modified
- **Destructive without confirmation**: `rm -rf`, `mkfs` without safeguards
- **Race conditions**: Time-of-check-to-time-of-use (TOCTOU) between `test -f` and `rm`
- **Temp files without `mktemp`**: Using predictable paths for temporary data
- **No trap for cleanup**: `trap cleanup EXIT` missing when script creates temp resources
- **`set -u` with default expansion**: Using `${VAR:-default}` correctly with `set -u`

## S11: NAMESPACE & ENV HYGIENE (7%)

**Source:** ops best practices, 12-factor app methodology

Check for:

- **Unprefixed environment variables**: Generic names (`DISK`, `USER`, `BASE_ROOT`) that collide with other tools or scripts
  ```bash
  # BAD: DISK collides with any other tool on the system
  DISK="${DISK:-/dev/sdc}"

  # GOOD: namespaced to this project
  MARIADB_DISK="${MARIADB_DISK:-/dev/sdc}"
  ```
- **Config file ownership conflicts**: Writing to shared config files (`containers.conf`, `storage.conf`, `registries.conf`) that other processes may also manage
  ```bash
  # RISKY: overwrites file another tool might maintain
  cp containers.conf ~/.config/containers/containers.conf

  # SAFER: use drop-in directories if supported, or warn
  # containers.conf: use [containers] sections in conf.d/ if podman supports it
  ```
- **XDG Base Directory non-compliance**: Hardcoding `~/.config`, `~/.local/share`, `~/.cache` instead of respecting `$XDG_CONFIG_HOME`, `$XDG_DATA_HOME`, `$XDG_CACHE_HOME` (XDG Base Directory Specification)
  ```bash
  # BAD: hardcoded, breaks in non-standard $HOME layouts
  config_dir="${HOME}/.config/myapp"

  # GOOD: XDG-compliant with fallback
  config_dir="${XDG_CONFIG_HOME:-${HOME}/.config}/myapp"
  data_dir="${XDG_DATA_HOME:-${HOME}/.local/share}/myapp"
  cache_dir="${XDG_CACHE_HOME:-${HOME}/.cache}/myapp"
  runtime_dir="${XDG_RUNTIME_DIR:-/tmp}/myapp"
  ```
- **Env leakage to child processes**: Exporting variables that should be local to the script
- **PATH pollution**: Appending to PATH without checking for duplicates
- **Default values vs fail-fast**: Using defaults for values that MUST be explicitly set
  ```bash
  # BAD: silently uses /dev/sdc which might be the wrong disk
  DISK="${DISK:-/dev/sdc}"

  # GOOD: force the operator to be explicit
  : "${MARIADB_DISK:?ERROR: MARIADB_DISK must be set (e.g., /dev/sdc)}"
  ```

## S12: SECURITY & INJECTION (5%)

**Source:** OWASP, CWE-78 (OS Command Injection), Google Shell Style Guide (Eval)

Check for:

- **`eval` usage**: Google explicitly says "eval should be avoided"
- **Unquoted variable in command position**: `$cmd "$args"` where `cmd` comes from user input
- **Shell injection via env vars**: Variables that get interpolated into `source`d or `eval`d contexts
  ```bash
  # VULNERABLE: if DATA_DIR contains $(rm -rf /), it executes when sourced
  echo "export DATA_DIR=\"${DATA_DIR}\"" >> env.sh
  source env.sh

  # SAFER: use printf %q for shell-safe quoting
  printf 'export DATA_DIR=%q\n' "${DATA_DIR}" >> env.sh

  # SAFEST: use a structured format (YAML/JSON) parsed by yq/jq
  ```
- **SUID/SGID on shell scripts**: Google explicitly bans this. Use `sudo` instead
- **Wildcard injection**: `rm *` in a directory where filenames could be `--no-preserve-root`
- **Tainted data in arithmetic**: `$(( user_input + 1 ))` — bash arithmetic can execute code
- **World-readable secrets**: `chmod 644` on files containing passwords/keys
