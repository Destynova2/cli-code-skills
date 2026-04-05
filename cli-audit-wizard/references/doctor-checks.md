# Doctor Checks -- Standard Categories

> **When to read:** During Step 3 of the audit workflow. Verify the wizard's doctor mode covers these check categories.

---

## Check categories

A doctor mode should cover these categories, adapted to the project's domain. Not all categories apply to every project -- but the auditor should verify that applicable ones are implemented.

### Schema

| Check | What it verifies | Expected behavior |
|-------|-----------------|-------------------|
| Config file exists | The config file is present at expected path | Fail if absent -> suggest `--setup` |
| Config parseable | File parses as valid TOML/YAML/JSON | Fail with line number and error |
| Schema version matches | Config schema version matches current binary | Warn if outdated -> suggest migration |
| Required keys present | All required config keys have values | Fail per missing key |
| Deprecated keys absent | No config keys from previous schema versions | Warn per deprecated key -> suggest removal |
| Value types valid | Each value matches expected type (int, string, bool, enum) | Fail per type mismatch |
| Value ranges valid | Numeric values within acceptable bounds (port 1-65535, timeout > 0) | Warn per out-of-range value |

### Connectivity

| Check | What it verifies | Expected behavior |
|-------|-----------------|-------------------|
| Backend reachable | HTTP probe on each configured backend | Warn per unreachable backend |
| DNS resolves | Each hostname in config resolves | Fail per unresolvable host |
| Port available | Configured listen ports are not in use | Fail per port conflict |
| Upstream latency | Response time from backends within threshold | Warn if > threshold |

### Credentials

| Check | What it verifies | Expected behavior |
|-------|-----------------|-------------------|
| Credential references present | Each required provider has a cred reference in config | Fail if missing -> trigger discovery cascade |
| Credential valid | API ping with stored token/key returns 200 | Fail if 401/403 -> re-trigger discovery for that provider |
| Credential expiry | OAuth tokens have expiry > 7 days | Warn if < 7 days -> suggest re-auth |
| No raw secrets in config | Config uses `_env`, `_cmd`, or `_keyring` refs, not raw values | Warn if raw secret detected -> suggest migration to env ref |
| Credential source accessible | Env var / keyring / password-manager CLI reachable | Fail if source missing (e.g., env var unset, 1Password locked) |
| Optional providers | Non-required providers not configured | Info only -> "Add OpenAI? [y/N]" |

**Credential discovery cascade order (for audit):**
1. Scan known paths (`~/.config/claude/`, `$ANTHROPIC_API_KEY`, OS keyring, `~/.netrc`, 1Password/Bitwarden CLI)
2. OAuth browser flow (with device-flow fallback for headless)
3. Manual API key paste (validate immediately)
4. Skip/defer (config written with placeholder, doctor flags it)

### Security

| Check | What it verifies | Expected behavior |
|-------|-----------------|-------------------|
| TLS cert valid | Certificate exists, not expired, matches hostname | Fail if expired or mismatched |
| TLS cert expiry | Certificate expiry date > 30 days | Warn if < 30 days |
| Secrets not in config | Config file doesn't contain raw secrets (env ref instead) | Warn if secrets detected |
| File permissions | Config file not world-readable | Warn if permissions too open |

### Filesystem

| Check | What it verifies | Expected behavior |
|-------|-----------------|-------------------|
| Log path writable | Configured log directory exists and is writable | Fail if not writable |
| Data dir exists | Configured data directories exist | Warn per missing dir -> suggest creation |
| Disk space | Minimum free space on config-relevant paths | Warn if < threshold |
| Signing key valid | ECDSA/RSA key present and loadable | Fail if missing or corrupt |

### Runtime

| Check | What it verifies | Expected behavior |
|-------|-----------------|-------------------|
| Service running | The main service process is alive | Fail if not running |
| Service healthy | Health endpoint returns 200 | Warn if unhealthy |
| Config matches running | Config on disk matches running config | Warn if diverged -> suggest reload |
| Version compatible | Binary version compatible with config schema | Fail if incompatible |

### CI/CD specific (server mode)

| Check | What it verifies | Expected behavior |
|-------|-----------------|-------------------|
| Caddy running | Reverse proxy is alive and serving | Fail if down |
| Caddy config matches | Caddyfile matches expected state | Warn if diverged |
| Container running | compose/k8s containers are up | Fail per stopped container |
| Network reachable | External clients can reach the service | Warn if not reachable |

---

## Doctor output format

Each check should produce a structured result:

```
[PASS] Schema: config parseable (/etc/app/config.toml)
[WARN] Security: TLS cert expires in 12 days -> renew with `certbot renew`
[FAIL] Connectivity: backend "mistral" unreachable (https://api.mistral.ai) -> check API key and network
```

## Exit codes

| Code | Meaning |
|------|---------|
| 0 | All checks pass |
| 1 | Warnings present, no failures |
| 2 | One or more failures |

## Reference implementations

| Tool | Doctor command | Notable pattern |
|------|--------------|-----------------|
| Homebrew | `brew doctor` | Modular check plugins, OS-specific variants |
| Rustup | `rustup check` | Version comparison, toolchain health |
| Helm | `helm lint` | Chart validation, template rendering check |
| Talos | `talosctl health` | etcd + API + node readiness + component status |
| kubectl | `kubectl cluster-info dump` | Comprehensive cluster state snapshot |
| step | `step ca health` | PKI-specific health checks |
