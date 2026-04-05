# Rotation Patterns -- Per-Provider Credential Renewal

> **When to read:** When a credential is flagged as expired, near-expiry, or due for rotation.
> Each pattern follows the molting rule: create new, deploy both, verify new, revoke old.
>
> **Staleness:** Verified 2026-04-05. Console URLs and CLI commands may change. The 4-step rotation pattern itself is durable; the specific commands per provider may need updating.

---

## The Universal 4-Step (Ecdysis Pattern)

```
1. CREATE   -- generate new credential (new exoskeleton forms)
2. DEPLOY   -- configure new alongside old (both valid simultaneously)
3. VERIFY   -- confirm new credential works in all services
4. REVOKE   -- revoke old credential (old exoskeleton shed)

NEVER: revoke -> create (soft-shell outage)
NEVER: create -> revoke -> deploy (gap in coverage)
```

---

## Anthropic (Claude)

### API Key Rotation
```
1. CREATE:  console.anthropic.com -> API Keys -> Create Key
2. DEPLOY:  export ANTHROPIC_API_KEY="sk-ant-api03-NEW..."
            (or update keychain / secret manager)
3. VERIFY:  curl -s https://api.anthropic.com/v1/messages \
              -H "x-api-key: $ANTHROPIC_API_KEY" \
              -H "anthropic-version: 2023-06-01" \
              -H "content-type: application/json" \
              -d '{"model":"claude-sonnet-4-20250514","max_tokens":1,"messages":[{"role":"user","content":"test"}]}'
            # 200 or 400 = key works
4. REVOKE:  console.anthropic.com -> API Keys -> Delete old key
```

### OAuth Token (Claude Code)
```
# OAuth tokens are managed by Claude Code automatically.
# To force re-auth:
claude auth logout
claude auth login
# Token stored in OS keychain, auto-refreshes.
```

### apiKeyHelper (Enterprise)
```
# The helper script is called automatically every 5 minutes.
# Rotation is handled by the script itself.
# To force refresh:
# Set CLAUDE_CODE_API_KEY_HELPER_TTL_MS=0 (force immediate refresh)
```

---

## OpenAI

### API Key Rotation
```
1. CREATE:  platform.openai.com -> API Keys -> Create new secret key
2. DEPLOY:  export OPENAI_API_KEY="sk-NEW..."
3. VERIFY:  curl -s https://api.openai.com/v1/models \
              -H "Authorization: Bearer $OPENAI_API_KEY" | head -1
            # Should return JSON with model list
4. REVOKE:  platform.openai.com -> API Keys -> Delete old key
```

### Codex CLI OAuth
```
# Codex uses OAuth with refresh tokens.
# To force re-auth:
codex auth logout
codex auth login
# Tokens stored in ~/.codex/auth.json or OS keychain.
# Access tokens auto-refresh via refresh token.
```

---

## GitHub

### Personal Access Token (Fine-Grained)
```
1. CREATE:  github.com -> Settings -> Developer settings -> Fine-grained tokens -> Generate
            (select minimum required repositories and permissions)
2. DEPLOY:  gh auth login --with-token < new-token.txt
            # or: export GITHUB_TOKEN="github_pat_NEW..."
3. VERIFY:  gh auth status
            # Shows: Logged in, token scopes, expiration
4. REVOKE:  github.com -> Settings -> Developer settings -> Delete old token
```

### Copilot CLI
```
# Uses device flow OAuth, managed by gh CLI.
gh auth login
# Token stored in OS keychain (service: "copilot-cli")
# Auto-refreshes via OAuth refresh token.
```

### gh CLI Token
```
# Stored in ~/.config/gh/hosts.yml
gh auth login                    # interactive re-auth
gh auth refresh                  # refresh existing token
gh auth status                   # verify
```

---

## Google (Gemini)

### API Key Rotation
```
1. CREATE:  aistudio.google.com/app/apikey -> Create API key
2. DEPLOY:  echo "GEMINI_API_KEY=AIzaNEW..." > ~/.gemini/.env
            chmod 0600 ~/.gemini/.env
3. VERIFY:  curl -s "https://generativelanguage.googleapis.com/v1/models?key=$GEMINI_API_KEY" | head -1
4. REVOKE:  aistudio.google.com -> API Keys -> Delete old key
```

### OAuth (Gemini CLI)
```
gemini auth logout
gemini auth login
# Token stored in OS keychain.
```

### Service Account (GCP / Vertex AI)
```
1. CREATE:  gcloud iam service-accounts keys create new-key.json \
              --iam-account=SA@PROJECT.iam.gserviceaccount.com
2. DEPLOY:  export GOOGLE_APPLICATION_CREDENTIALS="new-key.json"
3. VERIFY:  gcloud auth activate-service-account --key-file=new-key.json
            gcloud projects list  # should work
4. REVOKE:  gcloud iam service-accounts keys delete OLD_KEY_ID \
              --iam-account=SA@PROJECT.iam.gserviceaccount.com
```

---

## Mistral

### API Key Rotation
```
1. CREATE:  console.mistral.ai -> API Keys -> Create new key
2. DEPLOY:  export MISTRAL_API_KEY="NEW..."
            # or: echo "MISTRAL_API_KEY=NEW..." > ~/.vibe/.env && chmod 0600 ~/.vibe/.env
3. VERIFY:  curl -s https://api.mistral.ai/v1/models \
              -H "Authorization: Bearer $MISTRAL_API_KEY" | head -1
4. REVOKE:  console.mistral.ai -> API Keys -> Delete old key
```

---

## AWS

### IAM Access Keys
```
1. CREATE:  aws iam create-access-key --user-name myuser
2. DEPLOY:  aws configure set aws_access_key_id AKIANEW...
            aws configure set aws_secret_access_key NEW...
3. VERIFY:  aws sts get-caller-identity
            # Should return AccountId, UserId, Arn
4. REVOKE:  aws iam delete-access-key --access-key-id AKIAOLD... --user-name myuser
```

### SSO Session
```
# SSO tokens auto-expire (1-12 hours depending on IdP config).
# Re-auth:
aws sso login --profile my-profile
# Tokens cached in ~/.aws/sso/cache/*.json, auto-refresh where possible.
```

---

## Secret Managers -- Rotation Support

### 1Password
```bash
# Manual rotation:
op item edit "Anthropic API Key" password="sk-ant-api03-NEW..."
# Automated rotation via 1Password Secrets Automation:
# Configure rotation rules in 1Password dashboard.
# CLI reads current value:
op read "op://Development/Anthropic/api_key"
```

### Bitwarden
```bash
# Manual rotation:
bw edit item ITEM_ID --password="NEW..."
# Secrets Manager (automated):
bws secret edit SECRET_ID --value "NEW..."
```

### pass
```bash
# Manual rotation:
pass insert -f services/anthropic-api-key
# Enter new key, GPG-encrypts, git-commits automatically.
# History preserved in git:
pass git log services/anthropic-api-key
```

---

## Rotation Audit Checklist

For each credential rotation, verify:

```
[ ] New credential generated with correct scope (not broader)
[ ] New credential deployed to all services that use it
[ ] Both old and new credentials were valid simultaneously (no gap)
[ ] New credential verified via API ping before revoking old
[ ] Old credential revoked after verification
[ ] All references updated (env vars, config files, CI/CD secrets)
[ ] No copies of old credential remain (check CP-5: Scattered Secrets)
[ ] Rotation date recorded for tracking
```

---

## Provider Rotation Support Matrix

| Provider | Token type | Auto-expiry | Refresh token | Rotation API | Console rotation |
|----------|-----------|-------------|---------------|-------------|-----------------|
| Anthropic | API key | No | No | No | Manual (console) |
| Anthropic | OAuth (Claude Code) | Yes | Yes (auto) | N/A | `claude auth login` |
| OpenAI | API key | No | No | No | Manual (platform) |
| OpenAI | Codex OAuth | Yes | Yes (auto) | N/A | `codex auth login` |
| GitHub | Fine-grained PAT | Configurable | No | API (`/user/installations`) | Manual (settings) |
| GitHub | OAuth (gh CLI) | Yes | Yes (auto) | N/A | `gh auth refresh` |
| Google | API key | No | No | API (IAM) | Manual (console) |
| Google | OAuth | Yes | Yes (auto) | N/A | `gemini auth login` |
| Google | Service account | No | No | API (IAM) | `gcloud iam keys create` |
| Mistral | API key | No | No | No | Manual (console) |
| AWS | Access key | No | No | API (IAM) | `aws iam create-access-key` |
| AWS | SSO session | Yes (1-12h) | Yes (IdP) | N/A | `aws sso login` |
