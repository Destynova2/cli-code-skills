# Credential Registry -- Verified Paths from Source Code

> **When to read:** During credential discovery cascade implementation or audit (Step 2b).
> All paths verified from actual source code or official documentation.

> **Staleness warning:** Verified 2026-04-05. Stale after 90 days (2026-07-04).
> AI tools change credential paths frequently. If a known path is missing, ALWAYS fall back
> to discovery search (see "Resilient Discovery" section at bottom).
> Never trust a hardcoded path blindly -- verify it exists before reporting "not found."

---

## Environment Variables (check FIRST -- highest priority)

These block the interactive wizard entirely if set and valid (gh pattern).

| Provider | Env var(s) | Token prefix | Notes |
|----------|-----------|--------------|-------|
| **Anthropic** | `ANTHROPIC_API_KEY`, `ANTHROPIC_AUTH_TOKEN` | `sk-ant-api03-...` | AUTH_TOKEN takes precedence over API_KEY |
| **OpenAI** | `OPENAI_API_KEY`, `CODEX_API_KEY` | `sk-...` | CODEX_API_KEY for Codex CLI specifically |
| **GitHub** | `GITHUB_TOKEN`, `GH_TOKEN`, `COPILOT_GITHUB_TOKEN` | `gho_...` (OAuth), `github_pat_...` (fine-grained) | COPILOT_GITHUB_TOKEN > GH_TOKEN > GITHUB_TOKEN |
| **Google** | `GEMINI_API_KEY`, `GOOGLE_API_KEY`, `GOOGLE_APPLICATION_CREDENTIALS` | `AIza...` | GOOGLE_APPLICATION_CREDENTIALS = path to service account JSON |
| **Mistral** | `MISTRAL_API_KEY` | -- | |
| **OpenRouter** | `OPENROUTER_API_KEY` | -- | |
| **DeepSeek** | `DEEPSEEK_API_KEY` | -- | |
| **AWS** | `AWS_ACCESS_KEY_ID` + `AWS_SECRET_ACCESS_KEY` + `AWS_SESSION_TOKEN` | -- | Also `AWS_PROFILE`, `AWS_REGION` |
| **Ollama** | `OLLAMA_HOST` (default `127.0.0.1:11434`) | -- | Local-first, no API key |
| **Claude Code cloud** | `CLAUDE_CODE_USE_BEDROCK`, `CLAUDE_CODE_USE_VERTEX`, `CLAUDE_CODE_USE_FOUNDRY` | -- | Cloud provider routing flags |

---

## Credential Files on Disk (check second)

### Claude Code
**Source:** [claude-code auth docs](https://code.claude.com/docs/en/authentication)

| Platform | Path | Format |
|----------|------|--------|
| macOS | macOS Keychain (encrypted) | OS keyring |
| Linux | `~/.claude/.credentials.json` (mode `0600`) | JSON: `{ subscriptionType }` |
| Windows | Windows Credential Manager | OS keyring |
| Config | `~/.claude/settings.local.json` | JSON |
| Project config | `.claude/settings.local.json` | JSON |
| Override | `$CLAUDE_CONFIG_DIR` | -- |
| API key helper | `apiKeyHelper` setting, refreshes every 5 min or on 401 | Script path in config |
| Helper TTL | `$CLAUDE_CODE_API_KEY_HELPER_TTL_MS` | env var |

**Auth precedence:** Cloud provider env vars > `ANTHROPIC_AUTH_TOKEN` > `ANTHROPIC_API_KEY` > `apiKeyHelper` script > OAuth subscription

### OpenAI Codex CLI
**Source:** [codex-rs/login/src/auth/storage.rs](https://github.com/openai/codex/blob/main/codex-rs/login/src/auth/storage.rs)

| Item | Path / Value |
|------|-------------|
| Credential file | `~/.codex/auth.json` (mode `0600`) |
| Config dir override | `$CODEX_HOME` (default: `~/.codex/`) |
| Config file | `~/.codex/config.toml` |
| Keyring service | `"Codex Auth"` |
| Keyring key format | `"cli\|{sha256_hash_first_16_chars}"` (hash of codex_home path) |
| Credential store modes | `file` / `keyring` / `auto` (keyring with file fallback) / `ephemeral` |
| Config key | `cli_auth_credentials_store` in config.toml |
| OAuth callback | `localhost:1455` |
| File format | JSON: `{ auth_mode, OPENAI_API_KEY, tokens: {...}, last_refresh }` |

### GitHub Copilot
**Source:** [GitHub docs](https://docs.github.com/en/copilot), [DeepWiki](https://deepwiki.com/github/copilot-cli)

| Item | Path / Value |
|------|-------------|
| VS Code/Editor token | `~/.config/github-copilot/hosts.json` |
| VS Code alternate | `~/.config/github-copilot/apps.json` |
| XDG variant | `$XDG_CONFIG_HOME/github-copilot/hosts.json` |
| Windows | `%APPDATA%\Local\github-copilot\hosts.json` |
| Copilot CLI keyring | Service: `copilot-cli` |
| Copilot CLI config | `~/.copilot/config.json`, `~/.copilot/mcp-config.json` |
| gh CLI hosts | `~/.config/gh/hosts.yml` (YAML: `git_protocol`, `user`, `oauth_token`) |
| Device flow URL | `https://github.com/login/device` |

### Google Gemini CLI
**Source:** [gemini-cli/docs/get-started/authentication.md](https://github.com/google-gemini/gemini-cli/blob/main/docs/get-started/authentication.md)

| Item | Path / Value |
|------|-------------|
| Config dir | `~/.gemini/` |
| API key file | `~/.gemini/.env` (also searches up from CWD) |
| Config file | `~/.gemini/settings.json` |
| OAuth | OS keychain (macOS/Linux/Windows) |
| API key page | `https://aistudio.google.com/app/apikey` |

### Mistral Vibe
**Source:** [mistralai/mistral-vibe](https://github.com/mistralai/mistral-vibe)

| Item | Path / Value |
|------|-------------|
| API key file | `~/.vibe/.env` |
| Config file | `~/.vibe/config.toml` |

### Cursor IDE
**Source:** [Cursor forum](https://forum.cursor.com), [Cursor docs](https://cursor.com/docs/cli/reference/authentication)

| Item | Path / Value |
|------|-------------|
| macOS | `~/Library/Application Support/Cursor/` |
| Linux | `~/.config/Cursor/` |
| Windows | `%APPDATA%/Cursor/` |
| Settings DB | `{data_dir}/User/globalStorage/state.vscdb` (SQLite) |
| MCP config | `~/.cursor/mcp.json` |
| Auth tokens | OS keyring (via `keytar`); "weaker encryption" fallback |

### Continue.dev
**Source:** [continue/core/util/paths.ts](https://github.com/continuedev/continue/blob/main/core/util/paths.ts)

| Item | Path / Value |
|------|-------------|
| Config dir | `~/.continue/` (override: `$CONTINUE_GLOBAL_DIR`) |
| Config file | `~/.continue/config.yaml` (new), `~/.continue/config.json` (deprecated) |
| Secrets | `~/.continue/.env` with `${{ secrets.SECRET_NAME }}` syntax |
| File permissions | `0o600` via `chmodSync` on non-Windows |

### Aider
**Source:** [aider/main.py](https://github.com/Aider-AI/aider/blob/main/aider/main.py), [aider/onboarding.py](https://github.com/Aider-AI/aider/blob/main/aider/onboarding.py)

| Item | Path / Value |
|------|-------------|
| YAML config | `.aider.conf.yml` (searched: CWD, git root, `~/`) |
| .env file | `.env` in git root (default), or `--env <path>` |
| OAuth keys | `~/.aider/oauth-keys.env` (OpenRouter OAuth) |
| Model settings | `.aider.model.settings.yml` |
| Env var prefix | `AIDER_` (e.g., `AIDER_MODEL`) |

### AWS (Amazon Q / CodeWhisperer)
**Source:** [AWS CLI docs](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-files.html)

| Item | Path / Value |
|------|-------------|
| Config | `~/.aws/config` (INI) |
| Credentials | `~/.aws/credentials` (INI) |
| SSO cache | `~/.aws/sso/cache/*.json` (bearer + access tokens) |

### Ollama
**Source:** [ollama/envconfig/config.go](https://github.com/ollama/ollama/blob/main/envconfig/config.go)

| Item | Path / Value |
|------|-------------|
| Models dir | `$OLLAMA_MODELS` (default: `~/.ollama/models/`) |
| No credential file | Local-first, no API key storage |
| Auth flag | `OLLAMA_AUTH` (boolean, for remote registry auth) |

---

## OS Keychain Service Names (for probing)

```
macOS Keychain / Linux libsecret / Windows Credential Manager:
  "Codex Auth"          -> OpenAI Codex CLI
  "copilot-cli"         -> GitHub Copilot CLI
  (tool-specific)       -> Claude Code, Gemini CLI, Cursor
```

## Secret Managers

### 1Password CLI (`op`)
```bash
export OP_SERVICE_ACCOUNT_TOKEN="ops_..."    # service account
op run --env-file=.env -- your-command       # inject as env vars
op read "op://vault/item/field"              # read single secret
op plugin init                               # auto-inject for known CLIs
```

### Bitwarden CLI (`bw` / `bws`)
```bash
export BW_SESSION=$(bw unlock --raw)         # unlock vault
bw get password "item-name"                  # read single secret
export BWS_ACCESS_TOKEN="..."                # Secrets Manager
bws run -- your-command                      # inject as env vars
```

### pass (Unix Password Manager)
```bash
pass show services/openai-api-key            # GPG-encrypted
export OPENAI_API_KEY=$(pass services/openai-api-key)
# Storage: ~/.password-store/ (GPG files, git-tracked)
```

---

## Resilient Discovery (2-phase approach)

**Phase 1: Known paths (fast).** Check hardcoded paths from this registry. This covers 90% of cases when the registry is fresh.

**Phase 2: Pattern search (thorough).** If Phase 1 finds nothing for a provider you expect, search by pattern. This catches tools that moved their credential files since the registry was last updated.

### Phase 1 -- Known paths

```bash
# 1. Environment variables (immediate, highest signal)
for var in ANTHROPIC_API_KEY ANTHROPIC_AUTH_TOKEN \
           OPENAI_API_KEY CODEX_API_KEY \
           GITHUB_TOKEN GH_TOKEN COPILOT_GITHUB_TOKEN \
           GEMINI_API_KEY GOOGLE_API_KEY \
           MISTRAL_API_KEY OPENROUTER_API_KEY DEEPSEEK_API_KEY; do
  [ -n "${!var}" ] && echo "FOUND: $var"
done

# 2. Credential files (check existence + readability)
for f in ~/.claude/.credentials.json \
         ~/.codex/auth.json \
         ~/.config/github-copilot/hosts.json \
         ~/.config/github-copilot/apps.json \
         ~/.copilot/config.json \
         ~/.config/gh/hosts.yml \
         ~/.continue/config.yaml ~/.continue/.env \
         ~/.gemini/.env \
         ~/.vibe/.env \
         ~/.aider/oauth-keys.env \
         ~/.aws/credentials; do
  [ -r "$f" ] && echo "FOUND: $f"
done

# 3. OS Keychain (probe known service names)
# macOS:
security find-generic-password -s "Codex Auth" 2>/dev/null && echo "FOUND: Codex Auth (Keychain)"
security find-generic-password -s "copilot-cli" 2>/dev/null && echo "FOUND: copilot-cli (Keychain)"
# Linux:
secret-tool lookup service "Codex Auth" 2>/dev/null && echo "FOUND: Codex Auth (libsecret)"

# 4. Secret managers (check if CLI available)
command -v op   >/dev/null && echo "AVAILABLE: 1Password CLI"
command -v bw   >/dev/null && echo "AVAILABLE: Bitwarden CLI"
command -v bws  >/dev/null && echo "AVAILABLE: Bitwarden Secrets Manager"
command -v pass >/dev/null && echo "AVAILABLE: pass"
```

### Phase 2 -- Pattern search (fallback when known paths miss)

If a provider's known path doesn't exist, search by pattern. These patterns are more durable than exact paths because they match the *kind* of file, not the *exact location*.

```bash
# Search for credential-shaped files in common config dirs
# Look for JSON/TOML/YAML files with credential-related content
for dir in ~/.claude ~/.codex ~/.config/github-copilot ~/.copilot \
           ~/.continue ~/.gemini ~/.vibe ~/.aider ~/.aws ~/.config/gh; do
  [ -d "$dir" ] && find "$dir" -maxdepth 2 \
    \( -name '*.json' -o -name '*.yml' -o -name '*.yaml' \
       -o -name '*.toml' -o -name '*.env' -o -name 'credentials*' \
       -o -name 'auth*' -o -name 'hosts*' -o -name 'token*' \) \
    -readable 2>/dev/null
done

# Search for token patterns in found files (without printing the token)
# This confirms a file actually contains credentials
grep -rl 'sk-ant-\|sk-[a-zA-Z0-9]\{20,\}\|gho_\|github_pat_\|AIza[a-zA-Z0-9]\{20,\}\|AKIA[A-Z0-9]\{16\}' \
  ~/.claude/ ~/.codex/ ~/.config/github-copilot/ ~/.continue/ ~/.gemini/ ~/.vibe/ ~/.aider/ \
  2>/dev/null | while read f; do echo "CREDENTIAL_FILE: $f"; done

# Broader search: any .env or credential file in XDG config
find "${XDG_CONFIG_HOME:-$HOME/.config}" -maxdepth 3 \
  \( -name '.env' -o -name 'credentials*' -o -name 'auth.json' -o -name 'token*' \) \
  -readable 2>/dev/null
```

**Why Phase 2 matters:** Between registry updates, tools WILL move files. Claude Code could restructure `~/.claude/`, Codex could rename `auth.json`. Phase 2 catches these cases because it searches by pattern, not by exact path. The agent should report: "Known path ~/.claude/.credentials.json not found, but discovered credential file at ~/.claude/auth/oauth.json via pattern search."

---

## Maintenance Notes

> **Staleness policy:** This registry should be re-verified every 90 days.
> Fast-moving tools (Claude Code, Codex, Gemini CLI) may change paths between releases.
> Stable tools (AWS CLI, gh CLI, pass) change paths rarely.

**When to update:**
- A major AI tool changes its credential storage path
- A new widely-used AI CLI is released
- Token prefixes change (rare but happens)
- Phase 2 search consistently finds files that Phase 1 misses (registry is stale)

**How to verify a path is still current:**
1. Check the tool's GitHub repo for the auth/config module
2. Look for recent issues about credential storage changes
3. Install the tool fresh and run `strace` / `dtruss` to trace file access
3. Test on a fresh install
