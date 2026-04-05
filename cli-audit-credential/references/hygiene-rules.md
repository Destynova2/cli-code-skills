# Hygiene Rules -- Credential Storage & Handling Security

> **When to read:** During Step 4 of the audit workflow. Each rule maps to a check in the hygiene report.

---

## Storage Hierarchy (safest to riskiest)

| Rank | Method | Risk | When acceptable |
|------|--------|------|-----------------|
| 1 | **Hardware security module (HSM / TPM)** | Negligible | Production infrastructure |
| 2 | **Secret manager (1Password, Bitwarden, HashiCorp Vault)** | Very low | Any environment |
| 3 | **OS keychain (macOS Keychain, libsecret, Credential Manager)** | Low | Desktop/laptop interactive use |
| 4 | **Encrypted file (GPG, age, SOPS)** | Low | Headless servers, CI/CD with key management |
| 5 | **`pass` (GPG-encrypted flat files, git-tracked)** | Low-Medium | Personal credential management |
| 6 | **Environment variable** | Medium | CI/CD, containers, headless servers |
| 7 | **Plaintext file, mode 0600** | High | Only when keychain is genuinely unavailable |
| 8 | **Plaintext file, mode 0644+** | Critical | NEVER acceptable |
| 9 | **Config file with raw secret** | Critical | NEVER acceptable |
| 10 | **Git-tracked file with secret** | Critical | NEVER acceptable -- secret is permanent |

---

## Rules

### H-1: File Permission Rule

Credential files must be mode `0600` (owner read/write only) on Unix systems.

**Check:**
```bash
find ~/.claude ~/.codex ~/.config/github-copilot ~/.gemini ~/.vibe ~/.aider \
  ~/.continue ~/.aws -name '*.json' -o -name '*.env' -o -name '*.yml' -o -name '*.toml' \
  2>/dev/null | while read f; do
    perm=$(stat -c '%a' "$f" 2>/dev/null || stat -f '%Lp' "$f" 2>/dev/null)
    [ "$perm" != "600" ] && echo "FAIL: $f is mode $perm (should be 0600)"
done
```

**Fix:** `chmod 0600 <file>`

### H-2: No Raw Secrets in App Config

Application config files should reference credentials, not contain them.

**Good:**
```toml
[anthropic]
api_key_env = "ANTHROPIC_API_KEY"       # reads from env var
api_key_cmd = "op read op://vault/key"  # reads from 1Password
api_key_keyring = "anthropic-api"       # reads from OS keychain
```

**Bad:**
```toml
[anthropic]
api_key = "sk-ant-api03-..."            # raw secret in config file
```

**Check:**
```bash
# Scan config files for patterns that look like raw credentials
grep -rn 'sk-ant-api\|sk-[a-zA-Z0-9]\{30,\}\|ghp_\|gho_\|github_pat_\|AIza[a-zA-Z0-9]\{30,\}\|AKIA[A-Z0-9]\{16\}' \
  ~/.config/ ~/.claude/ ~/.codex/ ~/.continue/ ~/.aider/ 2>/dev/null
```

### H-3: Gitignore Enforcement

All credential-containing files must be in `.gitignore`.

**Files that must be gitignored:**
```
.env
.env.*
*.credentials*
*.secret*
~/.claude/.credentials.json
~/.codex/auth.json
~/.continue/.env
~/.aider/oauth-keys.env
~/.gemini/.env
~/.vibe/.env
```

**Check:**
```bash
# Are any .env files tracked?
git ls-files '*.env' '.env' '**/.env' 2>/dev/null
# Is .env in .gitignore?
grep -q '\.env' .gitignore 2>/dev/null || echo "FAIL: .env not in .gitignore"
```

### H-4: No Secrets in Git History

A secret removed from HEAD but present in git history is still compromised.

**Check:**
```bash
# Quick scan of recent history for credential patterns
git log --all -p --diff-filter=A -50 -- '*.env' '*.credentials*' 2>/dev/null | \
  grep -c 'sk-ant-\|sk-[a-zA-Z0-9]\{20,\}\|ghp_\|AKIA'
```

**Fix:** If secrets are found in history:
1. Rotate the compromised credential immediately (assume it's leaked)
2. Clean history with BFG Repo-Cleaner: `bfg --replace-text secrets.txt repo.git`
3. Force-push cleaned history (coordinate with team)
4. Notify affected services/providers

### H-5: No Secrets in Remote URLs

Git remote URLs should not contain embedded tokens.

**Check:**
```bash
git remote -v 2>/dev/null | grep -i 'token\|password\|secret\|ghp_\|gho_\|sk-'
```

**Fix:** Use credential helpers instead:
```bash
# Use gh CLI for GitHub auth
gh auth setup-git
# Or use OS credential helper
git config --global credential.helper osxkeychain   # macOS
git config --global credential.helper libsecret      # Linux
```

### H-6: Environment Variable Hygiene

Env vars are medium-risk. They're better than plaintext files but have known leakage paths.

**Leakage paths for env vars:**
- `/proc/<pid>/environ` (readable by same user on Linux)
- `ps e` (shows environment of running processes)
- Crash dumps and core files may contain env vars
- Child processes inherit all env vars (over-sharing)
- CI/CD logs may accidentally print env vars

**Mitigations:**
- Use env vars for CI/CD and containers (standard practice)
- Prefer keychain or secret manager for interactive desktop use
- Never `echo $API_KEY` or log env var values
- Use `env -i` or explicit env when spawning children

### H-7: Credential Scope Principle (Least Privilege)

Credentials should have the minimum scope needed.

| Scope level | When to use |
|------------|-------------|
| Read-only | Default for development, CI/CD reporting |
| Read-write | When the tool needs to create/modify resources |
| Admin | Never for automated tools. Human-only, temporary |

**Check:** For GitHub tokens, verify scopes:
```bash
# Check token scopes
curl -sI -H "Authorization: token $(gh auth token)" https://api.github.com | grep x-oauth-scopes
```

### H-8: Rotation Tracking

Indefinite API keys (Anthropic, OpenAI, Google) don't expire but should still be rotated.

**Organizational rotation policy:**
| Environment | Rotation interval | Enforcement |
|------------|-------------------|-------------|
| Production | 90 days | Mandatory (automated) |
| Staging | 90 days | Mandatory |
| Development | 180 days | Recommended |
| Personal | 365 days | Recommended |

**Tracking method:** Keep a `~/.credential-rotation.toml` or equivalent:
```toml
[anthropic]
created = "2026-01-15"
rotate_by = "2026-04-15"
source = "console.anthropic.com"

[openai]
created = "2026-03-01"
rotate_by = "2026-06-01"
source = "platform.openai.com"
```

---

## Quick Audit Script

```bash
#!/bin/bash
# credential-hygiene-check.sh -- quick hygiene scan

echo "=== File Permissions ==="
for f in ~/.claude/.credentials.json ~/.codex/auth.json \
         ~/.config/github-copilot/hosts.json ~/.gemini/.env \
         ~/.vibe/.env ~/.continue/.env ~/.aider/oauth-keys.env; do
  [ -f "$f" ] && stat -c '%a %n' "$f" 2>/dev/null
done

echo "=== Raw Secrets in Config ==="
grep -rln 'sk-ant-api\|sk-[a-zA-Z0-9]\{30,\}\|ghp_\|AKIA' \
  ~/.config/ ~/.claude/ ~/.codex/ ~/.continue/ ~/.aider/ 2>/dev/null

echo "=== Git-Tracked Secrets ==="
git ls-files '*.env' '.env' 2>/dev/null

echo "=== Remote URL Secrets ==="
git remote -v 2>/dev/null | grep -i 'token\|password\|ghp_\|gho_'

echo "=== .gitignore Check ==="
grep -q '\.env' .gitignore 2>/dev/null && echo "OK: .env in .gitignore" || echo "WARN: .env not in .gitignore"
```
