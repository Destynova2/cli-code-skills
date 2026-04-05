---
name: cli-audit-credential
description: >
  Discover, validate, and audit credential health across AI/dev tools.
  Scans tokens, checks expiry, detects raw secrets, verifies storage hygiene.
  Use on 'credential audit', 'token check', 'API key health', 'are my keys valid',
  'secret hygiene', 'rotate keys', 'credential scan', 'find my tokens', 'key inventory',
  'credential rotation', 'secret manager', 'keychain', '1Password', 'Bitwarden'.
argument-hint: "[provider-or-path]"
context: fork
agent: general-purpose
allowed-tools:
  - Read
  - Write
  - Grep
  - Glob
  - Bash
  - Agent
---

> **Optimization:** This skill uses on-demand loading. Heavy content lives in `references/` and is loaded only when needed.

> **Language rule:** Detect the project's primary language (from README, comments, docs, commit messages). Output your report in that language. If the project is bilingual, ask the user which language to use before proceeding.

> **Security rule:** This skill NEVER displays, logs, or includes raw credential values in its output. It reports presence, validity status, storage method, and expiry -- never the secret itself. When showing paths, mask values: `sk-ant-...****`.

# Audit Credential -- Credential Health Index (CHI)

Discover, validate, and audit the health of all AI/dev credentials on a machine or in a project. Goes beyond "is the key set?" to check validity, expiry, storage hygiene, rotation status, and secret isolation.

> "Every nucleated cell continuously displays peptide fragments on MHC molecules -- like a badge that says 'I'm authorized.'
> These badges are not permanent. If a cell stops displaying valid badges, natural killer cells destroy it."

## Natural laws mapped to credential rules

Each analogy maps to a concrete audit check:

| Source | Credential rule (testable) |
|--------|--------------------------|
| **Thymic selection** | 2-stage validation: format check (positive), then API ping (negative). Cheap filter first |
| **Blood-brain barrier** | Config references secrets, never contains them. Grep for raw tokens = barrier breach |
| **Complement cascade** | Check in order of cost: exists? -> permissions? -> format? -> API? Never call API if format fails |
| **Radioactive half-life** | Track time-to-expiry. Alert at 1 half-life remaining (7d/3d/1d thresholds) |
| **Faraday cage** | Secrets belong in keyring/vault. Plaintext file = outside the cage |
| **Heisenberg** | Audit tools must NEVER display the secret they're checking. Mask everything |
| **Entropy** (2nd law) | Unmanaged credentials drift. Periodic doctor is the energy that fights entropy |

> The full biological and physical analogies are in `references/analogies.md` (MHC turnover, vaccination, herd immunity, molting/ecdysis, and more).

## Credential Lifecycle

```
                         +---> ROTATE (new credential, revoke old)
                         |
DISCOVER -> VALIDATE -> USE -> MONITOR -> EXPIRE
   |           |                  |          |
   |           +-- FAIL: cascade  |          +-- RENEW (OAuth refresh)
   |               to next method |          +-- RE-DISCOVER (cascade)
   |                              |
   +-- INVENTORY (catalog all)    +-- DOCTOR (periodic health check)
```

Every credential passes through this lifecycle. The audit checks where each credential is in the cycle and whether the transitions are handled.

## Input

`$ARGUMENTS` is the target to audit:

- **Provider name** (`claude`, `openai`, `github`): audit only that provider's credentials
- **File path** (`~/.config/app/config.toml`): scan that file for embedded secrets
- **Directory** (`~/.config/`): scan all known credential locations under that path
- **`all`** or **empty**: full machine credential scan across all known providers

## Workflow

### Step 0 -- Discover credentials

Scan for all credentials on the machine. This is a passive read-only operation.

> Read `references/credential-registry.md` for the complete list of env vars, file paths, token prefixes, and keychain service names.

**Phase 0a -- Environment variables**

Check every known credential env var:
```bash
for var in ANTHROPIC_API_KEY ANTHROPIC_AUTH_TOKEN \
           OPENAI_API_KEY CODEX_API_KEY \
           GITHUB_TOKEN GH_TOKEN COPILOT_GITHUB_TOKEN \
           GEMINI_API_KEY GOOGLE_API_KEY GOOGLE_APPLICATION_CREDENTIALS \
           MISTRAL_API_KEY OPENROUTER_API_KEY DEEPSEEK_API_KEY \
           AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_SESSION_TOKEN \
           OLLAMA_HOST; do
  [ -n "${!var}" ] && echo "ENV: $var = [SET]"
done
```

For each found: record provider, source=env, value=masked.

**Phase 0b -- Credential files on disk (2-phase: known paths then pattern search)**

> Read `references/credential-registry.md` for the full list of known paths AND fallback search patterns.

First, check known paths from the registry (fast). Then, if a provider you expect is missing, run the pattern search fallback (thorough). The agent should use Glob and Grep tools rather than hardcoded bash -- the registry provides paths as guidance, not as a script to execute verbatim.

Key directories to scan: `~/.claude/`, `~/.codex/`, `~/.config/github-copilot/`, `~/.copilot/`, `~/.config/gh/`, `~/.continue/`, `~/.gemini/`, `~/.vibe/`, `~/.aider/`, `~/.aws/`

For each found: record provider, source=file, path, file permissions, format.

**Phase 0c -- OS Keychains**

Probe known keychain service names (service names from the registry: `"Codex Auth"`, `"copilot-cli"`, etc.). Use `security` on macOS, `secret-tool` on Linux.

For each found: record provider, source=keychain, service name.

**Phase 0d -- Secret managers**

Check for installed secret manager CLIs: `op` (1Password), `bw`/`bws` (Bitwarden), `pass`.

**Phase 0e -- Project-level scan**

If inside a git repo, scan for credentials embedded in project files:
- Search tracked files for token-shaped patterns (use Grep, not hardcoded bash)
- Pattern: strings matching `sk-ant-`, `sk-[a-zA-Z0-9]{20,}`, `ghp_`, `gho_`, `github_pat_`, `AIza[a-zA-Z0-9]{30,}`, `AKIA[A-Z0-9]{16}`
- Check `.env` files tracked in git (should be in `.gitignore`)
- Check `.git/config` for embedded tokens in remote URLs

### Step 1 -- Inventory

Build a credential inventory from all discoveries:

| # | Provider | Source | Location | Format | Status |
|---|----------|--------|----------|--------|--------|
| 1 | Anthropic | env | `$ANTHROPIC_API_KEY` | `sk-ant-...` | pending validation |
| 2 | OpenAI | file | `~/.codex/auth.json` | JSON (OAuth) | pending validation |
| 3 | GitHub | keychain | macOS Keychain `copilot-cli` | OAuth token | pending validation |
| ... | | | | | |

### Step 2 -- Validate (Thymic selection)

For each credential in the inventory, run the two-stage validation cascade:

**Stage 1 -- Positive selection (format check)**

Check if the token matches a known prefix pattern. These prefixes are stable (change rarely) but not permanent -- if a token doesn't match any known prefix, flag as UNKNOWN rather than FAIL and proceed to Stage 2 anyway.

| Provider | Known prefix (as of 2026-04) | Notes |
|----------|----------------------------|-------|
| Anthropic | `sk-ant-api03-` | Prefix has changed in the past (was `sk-ant-api02-`) |
| OpenAI | `sk-` (but not `sk-ant-`) | May also start with `sk-proj-` for project keys |
| GitHub (OAuth) | `gho_` | |
| GitHub (fine-grained PAT) | `github_pat_` | |
| GitHub (classic PAT) | `ghp_` (deprecated for Copilot) | |
| Google | `AIza` | |
| AWS | `AKIA` for access key IDs | |
| Unknown format | No prefix match | Do NOT reject -- proceed to Stage 2. Providers change prefixes. |

**Stage 2 -- Negative selection (API ping)**

For each credential, verify it actually works. Endpoints below are best-known as of 2026-04. If an endpoint returns a non-auth error (404, 5xx), the endpoint may have changed -- do NOT conclude the credential is invalid. Instead, flag as UNREACHABLE and suggest manual verification.

| Provider | Validation approach | Success | Notes |
|----------|-------------------|---------|-------|
| Anthropic | `POST /v1/messages` with minimal body | 200 or 400 | 400 = key valid, request malformed. Base: `api.anthropic.com` |
| OpenAI | `GET /v1/models` | 200 | Base: `api.openai.com` |
| GitHub | `GET /user` with Bearer token | 200 | Base: `api.github.com` |
| Google Gemini | `GET /v1/models?key=...` | 200 | Base: `generativelanguage.googleapis.com` |
| Mistral | `GET /v1/models` | 200 | Base: `api.mistral.ai` |
| AWS | `aws sts get-caller-identity` | JSON response | CLI command, not HTTP |
| Ollama | `GET /api/tags` | 200 | Base: `localhost:11434` (local) |

**Resilience rules for validation:**
- 401/403 = credential problem (EXPIRED or REVOKED)
- 404 = endpoint may have changed, not a credential problem (flag UNREACHABLE, suggest manual check)
- 5xx = server issue, not a credential problem (flag UNREACHABLE)
- Timeout/DNS fail = network issue (flag UNREACHABLE)
- Never conclude INVALID from a non-auth HTTP error

**Status after validation:**

| Status | Meaning | Action |
|--------|---------|--------|
| `VALID` | Format OK + API responds 200 | None |
| `EXPIRED` | Format OK + API responds 401/403 | Re-authenticate or rotate |
| `INVALID_FORMAT` | Token doesn't match expected prefix | Wrong token type, check provider |
| `UNREACHABLE` | Format OK + API timeout/DNS fail | Network issue, not credential issue |
| `INSUFFICIENT_SCOPE` | API responds 403 with scope error | Regenerate with correct scopes |
| `REVOKED` | API responds with explicit revocation message | Generate new credential |

### Step 3 -- Assess expiry (Half-life tracking)

For credentials that support expiry metadata:

| Provider | How to check expiry | Typical TTL |
|----------|-------------------|-------------|
| GitHub OAuth | Token response includes `expires_at` | 8 hours (device flow) |
| AWS SSO | `~/.aws/sso/cache/*.json` has `expiresAt` field | 1-12 hours |
| AWS session | `AWS_SESSION_TOKEN` + `expiration` in credential file | 1-36 hours |
| OAuth refresh tokens | Provider-specific introspection endpoint | 30-90 days |
| API keys (Anthropic, OpenAI, etc.) | No built-in expiry (manual rotation) | Indefinite -- flag as risk |

**Half-life alert thresholds:**

| Time to expiry | Alert level | Action |
|---------------|-------------|--------|
| > 7 days | `PASS` | None |
| 3-7 days | `INFO` | Plan rotation |
| 1-3 days | `WARN` | Rotate soon |
| < 24 hours | `CRITICAL` | Rotate now |
| Expired | `FAIL` | Re-authenticate |
| No expiry (indefinite API key) | `WARN` | Consider rotation policy |

### Step 4 -- Audit hygiene (Blood-brain barrier)

> Read `references/hygiene-rules.md` for the complete hygiene checklist.

For each credential found, check:

**4a -- Storage method**

| Method | Risk | Status |
|--------|------|--------|
| OS keychain (macOS Keychain, libsecret, Credential Manager) | Low | `PASS` |
| Secret manager (1Password, Bitwarden, pass) | Low | `PASS` |
| Environment variable (set in shell profile) | Medium | `INFO` -- acceptable for CI, warn for interactive |
| Encrypted file (mode 0600, encrypted content) | Medium | `INFO` |
| Plaintext file (mode 0600) | High | `WARN` -- acceptable if no keychain available |
| Plaintext file (mode 0644 or wider) | Critical | `FAIL` -- world-readable secret |
| Config file with raw key (not env ref) | Critical | `FAIL` -- secret committed to disk in app config |
| Git-tracked file with secret | Critical | `FAIL` -- secret in version history |

**4b -- Isolation check**

- Is the credential in a dedicated credential file (good) or embedded in app config (bad)?
- Does the app config use `_env`, `_cmd`, or `_keyring` references instead of raw values?
- Are `.env` files in `.gitignore`?
- Is the credential file excluded from backups (if applicable)?

**4c -- Git history scan**

```bash
# Check if any credential file is tracked
git ls-files --error-unmatch ~/.env .env **/.env 2>/dev/null

# Check for secrets in git history (last 50 commits)
git log --all -p -50 --diff-filter=A -- '*.env' '*.credentials*' '*.secret*' 2>/dev/null | \
  grep -c 'sk-ant-\|sk-[a-zA-Z0-9]\{20,\}\|ghp_\|AKIA'
```

**4d -- Permission check**

```bash
# All credential files should be mode 0600 (owner read/write only)
stat -c '%a %n' ~/.claude/.credentials.json ~/.codex/auth.json 2>/dev/null
# Any file with mode > 0600 = FAIL
```

### Step 5 -- Detect anti-patterns

> Read `references/anti-patterns.md` for the named credential anti-patterns.

### Step 6 -- Score and report

## Output Format

```markdown
# Credential Audit -- {machine-name or project-name}

**Date:** {date}
**Scope:** {all / provider / path}
**Credentials discovered:** {N}
**Providers covered:** {list}
**Credential Health Index:** {X}/100

## Credential Inventory

| # | Provider | Source | Location | Status | Expiry | Storage | Hygiene |
|---|----------|--------|----------|--------|--------|---------|---------|
| 1 | Anthropic | env | $ANTHROPIC_API_KEY | VALID | indefinite | env var | WARN |
| 2 | OpenAI | file | ~/.codex/auth.json | VALID | 45d | 0600 file | INFO |
| 3 | GitHub | keychain | macOS Keychain | VALID | 6h | keychain | PASS |
| 4 | Google | file | ~/.gemini/.env | EXPIRED | -3d | 0644 file | FAIL |
| 5 | Mistral | env | $MISTRAL_API_KEY | INVALID_FORMAT | -- | env var | FAIL |

## Validation Results

| Provider | Stage 1 (Format) | Stage 2 (API Ping) | Final Status |
|----------|-----------------|-------------------|-------------|
| Anthropic | PASS (sk-ant-...) | PASS (200) | VALID |
| Google | PASS (AIza...) | FAIL (401) | EXPIRED |
| Mistral | FAIL (no prefix match) | SKIPPED | INVALID_FORMAT |

## Expiry Tracking

| Provider | Time to Expiry | Half-Life Alert | Action |
|----------|---------------|-----------------|--------|
| Anthropic | indefinite | WARN | Consider rotation policy |
| OpenAI | 45 days | PASS | None |
| GitHub | 6 hours | CRITICAL | Will expire during workday |
| Google | -3 days | FAIL | Re-authenticate now |

## Hygiene Report

| Check | Status | Findings |
|-------|--------|----------|
| All secrets in keyring/vault | WARN | 2/5 in plaintext files |
| File permissions (0600) | FAIL | ~/.gemini/.env is 0644 |
| No raw secrets in config | PASS | All configs use env refs |
| No secrets in git history | PASS | Clean history |
| .env in .gitignore | PASS | All .env files excluded |

## Anti-Patterns Detected

| # | Pattern | Severity | Evidence | Fix |
|---|---------|----------|----------|-----|
| 1 | Immortal Key | Major | Anthropic key has no rotation policy | Set calendar reminder or use apiKeyHelper |
| 2 | World-Readable Secret | Critical | ~/.gemini/.env is mode 0644 | chmod 0600 ~/.gemini/.env |

## Recommendations

| # | Action | Priority | Impact | Command |
|---|--------|----------|--------|---------|
| 1 | Fix file permissions | Immediate | Security | `chmod 0600 ~/.gemini/.env` |
| 2 | Re-auth Google | Immediate | Functionality | `gemini auth login` |
| 3 | Rotate Mistral key | High | Functionality | Regenerate at console.mistral.ai |
| 4 | Move Anthropic key to keychain | Medium | Security | Use Claude Code OAuth login |
| 5 | Set rotation policy | Low | Hygiene | Add to quarterly rotation calendar |
```

## Scoring

```
chi = 100
  - (invalid_credentials x 15)        # EXPIRED, REVOKED, INVALID_FORMAT
  - (world_readable_files x 15)       # file permissions > 0600
  - (raw_secrets_in_config x 10)      # secrets not behind env/keyring ref
  - (secrets_in_git x 20)             # secrets in version history
  - (expiring_within_24h x 10)        # credentials about to expire
  - (no_expiry_no_rotation x 5)       # immortal keys without rotation plan
  - (plaintext_storage x 3)           # plaintext file when keyring is available
  - (missing_gitignore x 5)           # .env files not in .gitignore
  clamped to [0, 100]
```

| Score | Verdict |
|-------|---------|
| 80-100 | Healthy -- credentials valid, well-stored, rotation managed |
| 60-79 | Acceptable -- minor hygiene issues, all credentials functional |
| 40-59 | At risk -- expired credentials or storage hygiene problems |
| 20-39 | Degraded -- multiple invalid credentials, secrets exposed |
| 0-19 | Compromised -- secrets in git, world-readable files, widespread expiry |

## Rules for the Auditor

1. **Never display credential values.** Mask everything: `sk-ant-...****`, `gho_...****`. The audit reports presence and status, never the secret itself.
2. **Validation is read-only.** API pings should use minimal endpoints (list models, get user). Never create, modify, or delete resources during validation.
3. **Distinguish credential issues from network issues.** 401/403 = credential problem. Timeout/DNS fail = network problem. Don't conflate them.
4. **Indefinite API keys are a hygiene issue, not a failure.** Flag them as WARN, not FAIL. Many providers don't support expiry on API keys.
5. **OS keychain is the gold standard.** Always recommend keychain over plaintext file. But don't FAIL plaintext if keychain is genuinely unavailable (headless server, container).
6. **Git history is permanent.** A secret removed from HEAD but present in history is still compromised. Recommend `git filter-branch` or BFG Repo-Cleaner.
7. **Gotchas** -- read `../../gotchas.md` before producing output to avoid known mistakes.

## Integration with other cli-* skills

| Skill | Relationship |
|-------|-------------|
| `cli-audit-wizard` | Wizard audits the **UX flow** for setting up credentials. credential audits the **actual credentials** on disk |
| `cli-audit-code` | Code audit may find hardcoded secrets. credential audit finds them systematically |
| `cli-audit-shell` | Shell scripts may contain secrets in variables. credential audit checks shell profiles |
| `cli-forge-infra` | Infra configs reference secrets. credential audit verifies the references resolve |
| `cli-forge-pipeline` | CI/CD pipelines need secrets. credential audit checks CI env var hygiene |
| `cli-cycle` | Should call cli-audit-credential as part of the full project review |

## What this skill does NOT do

- **Does not generate credentials** -- it audits existing ones. Use provider dashboards to create new keys
- **Does not rotate credentials** -- it detects rotation needs. Use provider CLIs to rotate
- **Does not manage secret vaults** -- it checks that vaults are properly configured and referenced
- **Does not fix git history** -- it detects leaked secrets. Use BFG Repo-Cleaner to fix
- **Does not replace secret scanners** -- tools like truffleHog, gitleaks, detect-secrets are complementary
