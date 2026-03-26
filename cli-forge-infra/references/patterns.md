# Patterns & Strategies — SBC, NFD, Entropy, Patch Bankruptcy

> **When to read:** When debugging infra issues, auditing security/entropy, or applying the SBC/NFD strategies during Steps 2-5.

---

## Patch Bankruptcy Rule

When debugging or fixing infra, **count your failed attempts** on the same approach. Each consecutive failure doubles the probability that the approach itself is wrong, not the details.

```
Attempt  P(wrong approach)  Action
  1        50%              Try the fix
  2        75%              STOP. Read the official docs for the exact version.
  3        87%              MANDATORY STOP. Step back, check if a newer version
                            or a different approach eliminates the problem.
```

**The math:** if a fix has probability p=0.5 of working and you've failed N times, the probability the entire approach is wrong = 1 - p^N. After 2 failures: 75%. After 3: 87.5%. Continuing is statistically irrational.

### Real example (OpenBao init/unseal session)

```
Attempt 1: apk add curl          → no DNS in pod         FAIL
Attempt 2: rewrite with wget     → POST != PUT           FAIL
  ──── BANKRUPTCY LINE (P=75%) ──── should have stopped here ────
Attempt 3: wput() wrapper        → same POST problem     FAIL (wasted)
Attempt 4: bao CLI               → interrupted           FAIL (wasted)
Attempt 5: read the docs         → self-init found       SOLVED in 10min
```

Cost of attempts 3-4: ~20min. Cost of reading docs at attempt 2: ~10min. **Always cheaper to read docs after 2 failures.**

### When you hit bankruptcy

1. Stop patching. Read the official docs for the exact version in use.
2. Search: does a newer version solve this? (changelog/release notes)
3. Search: has someone else solved this? (GitHub issues, blog posts)
4. If the approach itself is wrong (imperative when declarative exists), redesign.

---

## Simplify-Before-Complexify (SBC)

When something doesn't work, the instinct is to add layers: a wrapper script, a sidecar container, an init container, a volume mount chain. **This is almost always wrong.** Each layer adds failure modes, makes debugging harder, and obscures the root cause.

**The rule: when stuck, REMOVE complexity before ADDING it.**

### The SBC Ladder (go DOWN, not up)

```
When a fix fails, go DOWN one rung before trying anything else:

  Current state (broken)
      ↓ REMOVE a layer, not add one
  Simplified state (still broken? now you know WHERE)
      ↓ REMOVE another layer
  Minimal reproduction (the bug is obvious here)
      ↓ FIX the actual root cause
  Add back layers one by one (each tested)
```

### Concrete steps

1. **Can you reproduce with just the service + CLI?**
   Skip the container, skip Terraform, skip the wrapper. Run the service locally (or `podman exec` into it) and test the exact command. If it works bare but not in the container, the problem is the container layer, not the service.

2. **Can you reproduce with the simplest config?**
   Strip the config to the absolute minimum (3-5 lines). Does it boot? Add lines back one by one until it breaks. The last line you added is the problem.

3. **Can you reproduce with one container?**
   Before debugging inter-container communication, test the service alone in a standalone container with `podman run`. No pods, no shared networks, no volumes. Does it work?

4. **Add layers back ONE AT A TIME.**
   Each layer gets tested before adding the next. Container works? Add the volume. Volume works? Add the network. Network works? Add the second container.

### Anti-patterns SBC prevents

| Anti-pattern | What happened | SBC would have done |
|-------------|---------------|-------------------|
| Alpine wrapper + env file + shared volume for vault-backend | 3 attempts, scratch image has no shell | Check: does vault-backend even NEED a token at startup? (No.) |
| `apk add curl` at runtime in container | Silent fail, no DNS | Check: what tools does the image actually have? |
| wget wrapper for PUT API calls | POST != PUT, 400 error | Check: does the service CLI handle this natively? |
| AppRole + wget login in wrapper script | Over-engineered, never tested bare | Check: read the vault-backend README first |

### Real example (vault-backend session)

```
WRONG approach (adding complexity):
  scratch image → add Alpine wrapper → add wait loop → add AppRole
  → add wget login → add binary download → STILL BROKEN

SBC approach (removing complexity):
  1. Read vault-backend README              → 5min
  2. "Creds come from TF client per request" → no token needed
  3. Use original scratch image, zero config → WORKS
```

Time wasted on wrong approach: ~30min. Time for SBC: ~5min.

---

## Network-First Debugging (NFD)

Network issues cause >50% of multi-service infra bugs. **Before debugging any service logic, verify the network model.**

### Step 1: Identify the network model

Every container runtime has a different network model. Get this wrong and nothing else matters.

```
| Runtime   | Model           | Inter-container comms              |
|-----------|-----------------|-------------------------------------|
| Podman pod | Shared localhost | All containers share 127.0.0.1      |
|           |                 | Container names do NOT resolve       |
|           |                 | Use localhost:<port> everywhere      |
| Docker Compose | Bridge network | Container names resolve via DNS |
|           |                 | Use service_name:<port>              |
| K8s pod   | Shared localhost | Same as Podman pod                  |
| K8s service | ClusterIP DNS | Use service_name.<ns>.svc:<port>     |
| Podman rootless | slirp4netns | Host ports mapped, no inter-pod DNS |
```

### Step 2: Verify before coding

Before writing any inter-service config, run these checks:

```bash
# Podman pod: can container A reach container B?
podman exec <container-A> wget -qO- http://127.0.0.1:<B-port>/health

# Does DNS resolve? (Podman pods: NO, it shouldn't)
podman exec <container-A> nslookup <container-B-name>

# What ports are actually listening?
podman exec <container-A> netstat -tlnp 2>/dev/null || \
podman exec <container-A> ss -tlnp 2>/dev/null || \
podman exec <container-A> cat /proc/net/tcp
```

### Step 3: Common traps

| Trap | Symptom | Fix |
|------|---------|-----|
| Container hostname in Podman pod | `dial tcp: lookup platform-gitea: no such host` | Use `127.0.0.1:<port>` |
| Wrong port (service changed default) | `connection refused` on expected port | Check service docs for current default port |
| Host-only bind (127.0.0.1) | Other containers can't reach it | Bind to `0.0.0.0` |
| Port conflict in shared namespace | Second service fails to bind | Change one service's port |
| rootless port < 1024 | Permission denied | Use port > 1024 or set sysctl |
| Service not ready yet | `connection refused` (race condition) | Health check loop, not sleep |

### Step 4: The 30-second network audit

For every multi-service config, verify this table BEFORE deploying:

```
| Service         | Listens on     | Accessed by        | Via                |
|-----------------|---------------|--------------------|--------------------|
| OpenBao         | 0.0.0.0:8200  | tofu-setup, vault-backend | 127.0.0.1:8200 |
| Gitea           | 0.0.0.0:3000  | tofu-setup, WP     | 127.0.0.1:3000     |
| Woodpecker      | 0.0.0.0:8000  | tofu-setup, Gitea webhook | 127.0.0.1:8000 |
| vault-backend   | 0.0.0.0:8080  | TF clients (external) | host:8080       |
```

If any "Via" column uses a container hostname in a Podman pod → **bug**.

### Real example (Gitea connection refused)

```
provider "gitea" { base_url = "http://platform-gitea:3000" }
                                    ^^^^^^^^^^^^^^^^
                                    Container hostname in Podman pod
                                    Resolves to 127.0.0.1 (sometimes)
                                    but DNS may not work → FAIL

Fix: base_url = "http://127.0.0.1:3000"
```

---

## Banned Entropy Patterns

### NEVER use

| Pattern | Why it's dangerous |
|---------|-------------------|
| `$RANDOM` in bash/sh | 15-bit PRNG, trivially predictable, NOT cryptographic |
| `shuf`, `sort -R` | Uses weak libc PRNG, not suitable for secrets |
| `date +%s` as seed/key | Predictable, guessable to the second |
| `head -c N /dev/random` (note: `/dev/random` with blocking) | Can block in low-entropy environments (containers, VMs), use `/dev/urandom` instead |
| `dd if=/dev/urandom \| md5sum` | Unnecessary hash step, loses entropy, MD5 is deprecated |
| `openssl rand` in shell scripts when the service has its own key generation | Unnecessary dependency; use the service's native generator |
| `echo "hardcoded-key-here"` | Self-explanatory |
| `uuidgen` as a secret | UUIDs are identifiers, not secrets — v4 uses weak PRNG on some systems |
| Hand-written key derivation in shell | Shell is not a crypto runtime |

### Correct Approaches (by priority)

**Priority 1 — Use the service's native key/secret generation:**
```
# OpenBao: let self-init generate its own keys
# cert-manager: let it generate its own CA
# Gitea: let it generate SECRET_KEY on first boot
# PostgreSQL: use CREATE ROLE ... PASSWORD (server-side)
```
The service knows its own crypto requirements. Let it handle them.

**Priority 2 — OS-level CSPRNG (`/dev/urandom`):**
```bash
# Generate 32 bytes of key material (hex-encoded)
head -c 32 /dev/urandom | xxd -p -c 64

# Generate 32 raw bytes (for file-based keys)
head -c 32 /dev/urandom > /path/to/key.bin

# Base64-encoded (for config files that need text)
head -c 32 /dev/urandom | base64
```
`/dev/urandom` is the correct source on Linux. It never blocks, and on any kernel >= 4.8 it is cryptographically secure even at early boot (uses ChaCha20 CSPRNG seeded from hardware entropy).

**Priority 3 — `openssl rand` (when openssl is available and service has no native generator):**
```bash
openssl rand -out /path/to/key.bin 32
openssl rand -hex 32
openssl rand -base64 32
```
Uses OpenSSL's CSPRNG, seeded from `/dev/urandom`. Acceptable but adds a dependency.

**Priority 4 — Language-native crypto libraries (in Terraform, Python, Go, etc.):**
```hcl
# Terraform: use random_password or random_bytes (backed by crypto/rand)
resource "random_password" "db_pass" {
  length  = 32
  special = true
}
```
```python
# Python: use secrets module (NOT random!)
import secrets
key = secrets.token_hex(32)
```
