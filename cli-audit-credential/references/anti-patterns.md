# Anti-Patterns -- Named Credential Failures

> **When to read:** During Step 5 of the audit workflow. Flag each detected anti-pattern with severity and evidence.

---

## CP-1: Immortal Key

**Severity:** Major

An API key with no expiry, no rotation schedule, and no monitoring. It was created once and will live forever -- or until it's compromised.

**Detection:**
- API key type (not OAuth token)
- No `created_at` or `rotate_by` metadata tracked
- No rotation policy documented
- Key is older than 180 days (if creation date can be inferred)

**Risk:** Indefinite keys accumulate compromise probability over time. Like Carbon-14, they decay slowly but inevitably.

**Fix:** Establish a rotation policy (90 days for prod, 180 for dev). Track creation dates. Set calendar reminders or automate rotation. If the provider supports short-lived tokens (OAuth), prefer those.

---

## CP-2: Barrier Breach

**Severity:** Critical

Raw credential value embedded directly in an application config file, committed to git, or visible in logs.

**Detection:**
- `grep` for token prefixes (`sk-ant-`, `sk-`, `ghp_`, `AIza`, `AKIA`) in config files
- Config files that contain `api_key = "sk-..."` instead of `api_key_env = "ENV_VAR"`
- Secrets in `docker-compose.yml`, Kubernetes manifests, or CI/CD config
- Token visible in `git log` history

**Risk:** Blood-brain barrier is breached. The secret is now in config files, backups, version history, and anywhere the file propagates.

**Fix:**
1. Rotate the compromised credential immediately (assume it's leaked)
2. Replace raw value with env var reference or secret manager reference
3. Clean git history if committed (BFG Repo-Cleaner)
4. Add pattern to `.gitignore` and pre-commit hooks

---

## CP-3: Soft-Shell Rotation

**Severity:** Major

Credential rotation where the old key is revoked before the new key is deployed and verified. The "soft-shell phase" (molting analogy) becomes a hard outage.

**Detection:**
- Rotation procedure that starts with "revoke old key"
- No overlap period between old and new credentials
- Rotation scripts that don't verify the new key before revoking the old

**Risk:** Service outage during rotation. If the new key is invalid, there's no fallback.

**Fix:** Always use the 4-step molting pattern:
1. Generate new credential
2. Deploy new alongside old (both valid)
3. Verify new works in all services
4. Revoke old only after verification

---

## CP-4: Scope Creep

**Severity:** Major

Credentials with broader permissions than needed. An admin token used for read-only operations. A full-access PAT used for a CI job that only needs repo read.

**Detection:**
- GitHub tokens with `admin:*` scopes used in CI
- AWS credentials with `AdministratorAccess` policy attached
- API keys with write access used in read-only contexts

**Risk:** If the credential leaks, the blast radius is maximum. A read-only token leak is an audit finding; an admin token leak is an incident.

**Fix:** Generate credentials with minimum required scope. Use separate credentials for different access levels. Audit scope after each use case change.

---

## CP-5: Scattered Secrets

**Severity:** Major

Same credential copied to multiple locations: env var, config file, CI/CD secrets, team shared doc, Slack message. No single source of truth for the credential.

**Detection:**
- Same token prefix found in multiple files/locations
- Team members have copies of the same API key
- Credential value appears in CI/CD config AND local config AND documentation

**Risk:** Rotation becomes impossible -- you can't find all the places the key is used. One copy survives and breaks when the key is rotated.

**Fix:** Single source of truth for each credential. All other locations reference the source:
- Source: OS keychain or secret manager
- References: env var (set from keychain at shell startup), config `_env` reference, CI/CD secret variable

---

## CP-6: Orphan Credential

**Severity:** Minor

A credential file or env var that references a service no longer in use. The token may still be valid but serves no purpose. It increases attack surface without providing value.

**Detection:**
- Credential file present but no corresponding tool installed
- Env var set for a provider not used in any project
- OAuth token for a service the user hasn't accessed in 6+ months

**Risk:** Forgotten credentials are unmonitored credentials. They may be compromised without anyone noticing because no one is using the service to trigger a failure.

**Fix:** Revoke and remove orphan credentials. If unsure whether a credential is still needed, check `~/.bash_history` or provider access logs for recent usage.

---

## CP-7: Provider Monoculture

**Severity:** Minor

All services authenticated with a single credential or single provider. If that provider goes down or revokes the key, everything fails simultaneously.

**Detection:**
- Same API key used across multiple projects/services
- All AI backends configured through a single provider
- No fallback provider configured

**Risk:** Single point of failure. Provider outage = total outage. Key revocation = total lockout.

**Fix:** Configure at least one fallback provider for critical services. Use separate credentials per project/service where possible.

---

## CP-8: Silent Expiry

**Severity:** Major

Credential expiry that produces no warning until the service fails. No monitoring, no alerting, no proactive renewal.

**Detection:**
- OAuth tokens with known expiry but no refresh mechanism configured
- No credential doctor/health check in CI/CD pipeline
- No alerting on 401/403 errors from credential-dependent services

**Risk:** Service fails at the worst possible time (Friday evening, during a demo, in production).

**Fix:**
- Implement credential doctor (this skill) as a periodic check
- Configure OAuth refresh token flows where supported
- Add 401/403 alerting to service monitoring
- Track expiry dates and alert at 7d, 3d, 1d thresholds

---

## CP-9: Ambient Authority

**Severity:** Major

Tool automatically uses credentials from the environment without explicit configuration. The user doesn't know which credential is being used or where it came from.

**Detection:**
- Tool uses `$OPENAI_API_KEY` without telling the user
- Multiple tools share the same env var but expect different scopes
- Tool inherits credentials from parent process unexpectedly

**Examples:**
- A CI job inherits `$GITHUB_TOKEN` from the runner, uses it for an unintended purpose
- A user sets `$OPENAI_API_KEY` for one tool, another tool silently uses it

**Fix:** Tools should log which credential they're using and where it came from (env var, file, keychain). The `gh` pattern: "Using token from $GH_TOKEN environment variable."

---

## Summary

| # | Anti-Pattern | Severity | Analogy | Core Issue |
|---|-------------|----------|---------|------------|
| CP-1 | Immortal Key | Major | Carbon-14 slow decay | No rotation policy |
| CP-2 | Barrier Breach | Critical | BBB breakdown | Raw secret in config/git |
| CP-3 | Soft-Shell Rotation | Major | Premature molting | Old revoked before new verified |
| CP-4 | Scope Creep | Major | Autoimmune disease | Over-privileged credentials |
| CP-5 | Scattered Secrets | Major | Metastasis | Same key in multiple locations |
| CP-6 | Orphan Credential | Minor | Vestigial organ | Unused but still valid |
| CP-7 | Provider Monoculture | Minor | Monoculture crop | Single point of failure |
| CP-8 | Silent Expiry | Major | Undetected cancer | No monitoring/alerting on expiry |
| CP-9 | Ambient Authority | Major | Autoimmune confusion | Tool uses unexpected credential |
