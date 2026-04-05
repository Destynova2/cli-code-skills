# Reference Flows -- Real-World Best Practices

> **When to read:** When auditing a wizard's credential flow, re-run behavior, or doctor mode. Compare against these proven implementations.
>
> **Staleness:** Verified 2026-04-05. CLI auth flows evolve (Vercel consolidated 5 methods to 1 in 2025). The patterns (env-first, browser-fallback, device-flow) are durable; specific flags and URLs may drift.

---

## Auth Discovery -- The Universal Three-Tier Hierarchy

Every well-designed CLI follows the same order. Deviation is an anti-pattern.

```
1. ENV VAR CHECK (before any prompts)
   +-- $TOOL_TOKEN, $API_KEY, etc.
   +-- Found + valid? -> use it, skip wizard entirely
   +-- Found + invalid? -> warn, continue to next tier

2. BROWSER / OAUTH (default interactive)
   +-- Device code flow (RFC 8628) -- modern standard
   +-- Or browser redirect with localhost callback
   +-- Headless fallback: show URL + code for manual entry
   +-- Token stored in OS keychain or config file

3. TOKEN PASTE (last resort)
   +-- "Paste your API key: ********"
   +-- Validate immediately (API ping)
   +-- Store as env var reference, not raw value
```

### Concrete implementations

**`gh auth login`** -- The gold standard interactive auth:
1. Checks `GH_TOKEN` / `GITHUB_TOKEN` env var -> blocks wizard if set
2. Host selection (GitHub.com / Enterprise)
3. Git protocol (HTTPS / SSH) -> if SSH, auto-detects existing keys, offers to generate
4. Auth method: "Login with web browser" / "Paste token"
5. Browser path: shows one-time device code, opens browser, polls until authorized
6. Token stored in OS keychain; plaintext fallback with warning

**`gcloud auth login`** -- Three levels of headless support:
1. Default: opens browser automatically
2. `--no-launch-browser`: shows URL, user pastes back auth code
3. `--no-browser`: outputs a `gcloud auth login --remote-bootstrap="..."` command to run on another machine with a browser

**`aws configure sso`** -- Progressive with auto-skip:
1. Browser opens for SSO portal auth (device code fallback with `--use-device-code`)
2. Account selection -- auto-skipped if only 1 account
3. Role selection -- auto-skipped if only 1 role
4. Profile name with smart default

**`railway login`** -- Minimal modern flow:
1. Checks `RAILWAY_TOKEN` env var -> skips if set
2. Opens browser -> done
3. `--browserless`: shows pairing code + URL

**`vercel login`** -- Simplified over time:
- Consolidated 5 auth methods (email, GitHub, GitLab, Bitbucket, OOB) down to 1 device flow
- Lesson: **fewer auth paths = less confusion**

**`supabase login`** -- Simplest pattern (token-paste only):
1. Shows link to dashboard token page
2. "Paste your access token: ..."
3. Stores in OS keyring; falls back to `~/.supabase/access-token`

### Auth anti-patterns observed in the wild

| Anti-pattern | Example | Impact |
|-------------|---------|--------|
| Browser-only auth, no headless fallback | (various) | Broken on servers, CI, SSH, containers |
| Opaque error on auth failure | "Bad credentials" with no guidance | Users spend hours debugging |
| No env var override for CI | (various) | Cannot automate |
| Breaking auth change silently | Azure CLI mandatory MFA enforcement (2025) | Scripts break overnight |
| Token format ambiguity | GitHub PAT classic vs fine-grained | Users create wrong token type |

---

## Re-Run Behavior -- Four Patterns

### Pattern 1: Prefill + Merge (RECOMMENDED)

**Example: `npm init` when `package.json` exists**
- Reads existing config as starting template
- Prefills all prompts with current values
- Press Enter to keep existing value
- Only overwrites fields user explicitly changes
- `npm init -y` with existing file: updates only blank fields

**This is the target behavior for wizard re-runs.**

### Pattern 2: Idempotent + Migration Flags

**Example: `terraform init` re-run**
- No changes -> no-op, safe to re-run
- Backend changed -> offers `--reconfigure` (discard) or `--migrate-state` (copy)
- Interactive confirmation before destructive migration
- `-force-copy` suppresses prompts for CI

**Best for stateful systems where re-init has real consequences.**

### Pattern 3: Hard Block (AVOID unless scaffolding)

**Example: `cargo init` when `Cargo.toml` exists**
- Flat refusal: "cannot be run on existing Cargo packages"
- Intentional for project scaffolding (partial overwrite would corrupt)
- Acceptable ONLY when the wizard creates complex file structures

### Pattern 4: Warn + Confirm

**Example: `eslint --init` (legacy)**
- Detects existing config
- "An eslint config already exists. Overwrite? [y/N]"
- Binary choice: keep all or replace all

**Worst of both worlds -- no partial edit, no merge.**

### Re-run audit checklist

```
[ ] Wizard detects existing config file
[ ] Current values pre-fill prompts (not blank)
[ ] User can press Enter to keep each value unchanged
[ ] Only changed values appear in the diff
[ ] Unchanged config produces no diff (idempotent)
[ ] --edit [section] opens only that section, pre-filled
[ ] No data loss on re-run (never overwrites unrelated sections)
```

---

## Doctor Mode -- The `flutter doctor` Standard

### Output format benchmark

```
[✓] Category Name (details)
[!] Category Name
    ✗ Specific issue found
      Fix: exact command to run
[✗] Category Name
    ✗ Critical problem
      Fix: exact command to run
    ✗ Another problem
      Fix: link to documentation

! Doctor found issues in N categories.
```

### Three status levels

| Symbol | Meaning | User action |
|--------|---------|-------------|
| `[✓]` | Pass | None |
| `[!]` | Warning | Optional fix, system works |
| `[✗]` | Failure | Must fix for system to work |

### Doctor design rules (from flutter doctor, brew doctor, rustup check)

1. **Summary first, details on demand.** Default = one line per category. `-v` reveals sub-checks.
2. **Every failure includes a fix.** Not "X is wrong" but "X is wrong. Run `command` to fix."
3. **Independent checks.** One failure never blocks other checks from running.
4. **Machine-readable output.** `--json` flag for CI integration.
5. **Exit code = severity.** 0 = all pass, 1 = warnings, 2 = failures.
6. **Disclaimer for warnings.** brew doctor: "If everything works, please don't worry." Prevents alarm fatigue.

### Doctor implementations compared

| Tool | Style | Actionability | Machine-readable |
|------|-------|---------------|-----------------|
| `flutter doctor` | Symbols + categories + fix commands | Excellent | `flutter doctor --verbose` |
| `brew doctor` | Prose warnings | Medium | No (stderr only) |
| `rustup check` | One line per toolchain | Minimal | No |
| `npx envinfo` | Structured data dump | Info only | `--json` |
| `next info` | Issue-report generator | Info only | Copy-paste format |

---

## Progressive Disclosure -- Zero-Config as Ideal

### Tailscale -- The "no wizard" wizard

Entire setup: `tailscale up` -> browser opens -> sign in -> done.
Zero configuration questions. Behind the scenes: WireGuard keypair, IP assignment, NAT traversal, peer discovery, key rotation -- all automatic.

Complexity is opt-in via flags:
- `--advertise-routes` (expose subnet)
- `--exit-node` (VPN mode)
- ACL policies via web dashboard only
- Custom DERP servers via config file (power users)

### Caddy -- Zero-config HTTPS

Minimum config: `example.com { reverse_proxy localhost:3000 }`
Caddy auto-provisions TLS from Let's Encrypt, sets up redirects, auto-renews. Zero questions.

Progressive layers:
1. Domain in Caddyfile = automatic HTTPS
2. `tls` directive = customize cert settings
3. `tls internal` = self-signed for dev
4. Custom ACME, DNS challenge = advanced
5. JSON API = programmatic control

### The principle

> The best wizard is no wizard. If you must ask, ask one thing. If you must ask more, ask three things max. Everything else is a flag or a default.

### Auto-skip rule

When a prompt has only one valid answer, skip it silently. AWS SSO auto-selects when there is one account. `gh` auto-detects SSH keys. Never ask a question with a predetermined answer.

---

## Sources

- gh auth login: https://cli.github.com/manual/gh_auth_login
- gcloud auth login: https://docs.google.com/sdk/gcloud/reference/auth/login
- AWS CLI SSO: https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-sso.html
- railway login: https://docs.railway.com/cli/login
- vercel login: https://vercel.com/docs/cli/login
- wrangler login: https://blog.cloudflare.com/wrangler-oauth/
- supabase login: https://supabase.com/docs/reference/cli/supabase-login
- fly auth login: https://fly.io/docs/flyctl/auth-login/
- npm init: https://docs.npmjs.com/cli/v11/commands/npm-init/
- terraform init: https://developer.hashicorp.com/terraform/cli/commands/init
- flutter doctor: https://docs.flutter.dev/get-started/install
- brew doctor: https://docs.brew.sh/rubydoc/Homebrew/Diagnostic/Checks.html
- Tailscale quickstart: https://tailscale.com/docs/how-to/quickstart
- Caddy automatic HTTPS: https://caddyserver.com/docs/automatic-https
- The Wizard Anti-Pattern: https://stef.thewalter.net/installer-anti-pattern.html
- RFC 8628 (Device Authorization Grant): https://datatracker.ietf.org/doc/html/rfc8628
