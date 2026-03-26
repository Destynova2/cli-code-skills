# Scoring Systems — IDS, DHS & Infrastructure Health Scorecard

> **When to read:** During Step 7 (Periodic Review) or when scoring infra code quality, dependency health, or imperative debt.

---

## Imperative Debt Score (IDS)

Count imperative patterns in infra code. Each is a "debt point" — a place where things can silently break, a tool can be missing, or a race condition can occur.

### Scoring per file

| Pattern | Points | Why |
|---------|--------|-----|
| `curl` / `wget` raw API call | 2 | Tool may not exist; HTTP method may be wrong |
| `sleep N` / `until ... do sleep` | 2 | Race condition, arbitrary timeout |
| `apk add` / `apt install` at runtime | 3 | Needs network + DNS, may fail silently |
| `\|\| true` / `2>/dev/null` | 2 | Swallowed error, silent failure |
| `grep \| sed \| awk \| cut` pipeline | 1 | Fragile text parsing, breaks on format change |
| `$RANDOM` / `shuf` for secrets | 5 | Security vulnerability (see entropy patterns) |
| Manual JSON construction in shell | 2 | Quote escaping hell, breaks on special chars |
| `chmod` / `chown` in entrypoint | 1 | UID mismatch between image and runtime |
| Hardcoded port/host in script | 1 | Breaks when service config changes |

### Thresholds

```
IDS 0-3   → GREEN   Minimal scripting, acceptable
IDS 4-8   → YELLOW  Consider declarative alternatives
IDS 9-15  → ORANGE  Refactor: replace with TF/Helm/self-init
IDS 16+   → RED     Script is doing too much. Split or rewrite.
```

### Real example — old `setup.sh` OpenBao section

```
curl -sf -X POST ... login/bootstrap-admin   → 2 (curl API call)
curl -sf -X POST ... token/create             → 2 (curl API call) x3 = 6
printf '%s' "$TOKEN" > file                   → 0 (fine)
gen_hex() { head -c ... /dev/urandom ... }    → 0 (correct entropy)
curl -sf -X POST ... secret/data/cluster      → 2 (curl API call) x2 = 4
until curl ... health; do sleep 2; done       → 2 (sleep loop)
"$CI_PASSWORD" in -d JSON                     → 2 (manual JSON)
                                        TOTAL → 16 = RED
```

Verdict: this entire block should be replaced by Terraform vault provider or OpenBao self-init `eval_source`.

---

## Dependency Health Score (DHS)

Before relying on any third-party tool, score it:

```
DHS = stars × recency × maintenance

Where:
  stars     = GitHub stars (log scale: <10=0.1, <50=0.3, <200=0.6, <1K=0.8, 1K+=1.0)
  recency   = last commit (this month=1.0, <3mo=0.8, <6mo=0.5, <1y=0.3, >1y=0.1)
  maintenance = has CI + releases + responds to issues (yes=1.0, partial=0.5, no=0.2)

DHS < 0.1 → DO NOT USE — find alternative or build in-house
DHS 0.1-0.3 → RISKY — have a fallback plan, pin version
DHS 0.3-0.7 → ACCEPTABLE — use with version pinning
DHS > 0.7 → GOOD — standard dependency
```

### Real example — `gherynos/vault-backend`

```
stars     = 5     → 0.1
recency   = Jan   → 0.3 (2 months ago)
maintenance = no CI, no releases → 0.2
DHS = 0.1 × 0.3 × 0.2 = 0.006 → DO NOT USE
```

Action: search for alternatives (`frieser/terraform-vault-backend`, `ei-grad/tfstate-http-backend-vault`) or use TF native backend (S3/consul/local).

---

## Infrastructure Health Scorecard

Use during periodic review (Step 7) when invoked on a directory without a specific issue.

```
| Area              | Status | Notes |
|-------------------|--------|-------|
| Version currency  | 🔴/🟡/🟢 | Are services on latest stable? Check changelogs. |
| Config simplicity | 🔴/🟡/🟢 | IDS score per file. Any script > 8 points? |
| Security posture  | 🔴/🟡/🟢 | Secrets from CSPRNG? TLS everywhere? No $RANDOM? |
| Entropy quality   | 🔴/🟡/🟢 | All keys from proper sources? Rotation planned? |
| Network model     | 🔴/🟡/🟢 | 30-second audit table filled? Hostnames correct? |
| Bootstrap reliability | 🔴/🟡/🟢 | Does `make up` work from clean state? |
| Dependency health | 🔴/🟡/🟢 | DHS score per dep. Any < 0.1? |
| Inline docs       | 🔴/🟡/🟢 | Every stanza commented? Max 80 chars? |
```

For each non-green item, provide:
1. What's wrong
2. What the fix looks like
3. Effort estimate (trivial / small / medium / large)
4. Priority (do now / next sprint / backlog)
