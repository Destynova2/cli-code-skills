# Analogies -- Biological & Physical Models for Credential Management

> **When to read:** When explaining credential audit findings, or when building intuition for the right fix.

---

## I. BIOLOGY -- The Immune Credential System

### Token Expiry -> MHC Molecule Turnover

Every nucleated cell in the body continuously displays peptide fragments on MHC class I molecules -- name badges that say "I'm self, I'm authorized." These badges are not permanent:

| MHC Biology | Credential Management |
|-------------|----------------------|
| MHC displays peptides = "I am authorized" | Token proves service identity |
| Rapid turnover on immature dendritic cells (hours) | Short-lived tokens during setup/dev |
| Long half-life on activated memory cells (100+ hours) | Longer-lived tokens in stable production |
| Missing MHC = cell killed by NK cells | Expired token = service rejected by gateway |
| MHC restriction (only certain T-cells match) | Scoped tokens (read-only, namespace-limited) |
| MHC class I (all cells) vs class II (antigen-presenting only) | User tokens (broad) vs service tokens (specialized) |

**Rule:** Every credential should have an explicit TTL. Default to short (dev = 1h, staging = 24h, prod = 7d with rotation). A cell without a valid MHC badge gets killed -- a service without a valid token should too.

### Credential Validation -> Thymic Selection

T-cells undergo a brutal two-stage selection in the thymus:

1. **Positive selection** (cortex): Can this T-cell recognize self-MHC at all? If not, it's useless -- apoptosis (death by neglect). ~90% of T-cells die here.
2. **Negative selection** (medulla): Does this T-cell bind self-MHC too strongly? If yes, it would attack the body's own cells -- apoptosis (death by instruction). ~5% more die here.

Only ~2-5% of T-cells survive both filters and enter circulation.

Credential validation should work the same way:

| Thymic Stage | Credential Check | Failure Mode |
|-------------|-----------------|--------------|
| **Positive selection** | Format check: does the token match expected prefix/pattern? (`sk-ant-`, `gho_`, `AIza`) | Wrong token type, corrupted, wrong provider |
| **Negative selection** | API ping: does the token actually authenticate? Is the scope correct? Does it NOT have dangerous permissions? | Expired, revoked, overprivileged |

**Rule:** Validate in two stages, always in order. A token that fails format check should never hit the API (positive selection eliminates 90% of bad inputs cheaply). A token that passes format but fails API = expired/revoked (negative selection catches the rest).

### Secret Isolation -> Blood-Brain Barrier

The blood-brain barrier (BBB) is formed by tight junctions between endothelial cells lining brain capillaries. Most molecules in the bloodstream -- including toxins, pathogens, and large proteins -- cannot cross into the brain. The brain receives only what it specifically transports through receptor-mediated mechanisms.

| BBB Biology | Secret Management |
|-------------|-------------------|
| Tight junctions between cells | Config file references secrets, never contains them |
| Only specific receptors allow transport | Only `_env`, `_cmd`, `_keyring` reference types cross the barrier |
| Toxins in blood don't reach the brain | Raw secrets in env don't leak into config files, logs, or git |
| BBB breakdown = neurological disease | Secret isolation breach = credential leak |
| Astrocyte foot processes reinforce the barrier | `.gitignore`, file permissions, audit tools reinforce isolation |

**Rule:** Secrets must never cross the barrier into application config files. The config file should contain a reference (`api_key_env = "ANTHROPIC_API_KEY"`), never the value. The barrier is enforced by `.gitignore`, file permissions (0600), and this audit skill.

### Validation Cascade -> Complement System

The complement system is a cascade of 30+ serum proteins that activate in sequence. Each step amplifies the signal and recruits the next step. The cascade has three pathways (classical, lectin, alternative) but they all converge on the same final steps: opsonization (tagging), inflammation, and membrane attack (killing).

Credential validation follows the same cascade pattern:

```
EXISTS? -> READABLE? -> CORRECT FORMAT? -> API RESPONDS? -> SCOPES OK? -> NOT EXPIRING?
   |           |              |                 |               |              |
   v           v              v                 v               v              v
 MISSING    PERMISSION     INVALID           EXPIRED/        INSUFFICIENT   EXPIRING
            DENIED         FORMAT            REVOKED          SCOPE          SOON
```

Each step is cheap and filters out a class of problems before the next more expensive check. Like complement, the cascade amplifies: finding one invalid credential triggers a deeper scan of related credentials.

**Rule:** Always run checks in order of cost: existence (free) -> permissions (stat) -> format (regex) -> API ping (network). Never call the API if the format check failed.

### Graceful Rotation -> Molting (Ecdysis)

Arthropods (insects, crustaceans) grow by molting: they secrete a new exoskeleton under the old one, then shed the old one. During the brief vulnerable period between shedding and hardening, the animal is soft and at risk.

Credential rotation follows the same pattern:
1. Generate new credential (new exoskeleton forms under the old)
2. Deploy new credential alongside old (both are valid temporarily)
3. Verify new credential works (new exoskeleton hardens)
4. Revoke old credential (old exoskeleton shed)
5. Brief vulnerable window during transition (soft-shell phase)

**Rule:** Rotation must be a 4-step process, never a 2-step (revoke then create). The old credential must remain valid until the new one is verified. The "soft-shell" phase should be minimized but acknowledged.

### Periodic Audit -> Immune Surveillance

The immune system doesn't wait for infection. Natural Killer cells continuously patrol the body, checking every cell's MHC display. Dendritic cells continuously sample the environment for foreign antigens. This is immune surveillance -- constant, passive monitoring.

**Rule:** Credential audits should run periodically, not just when something breaks. Integrate `cli-audit-credential` into `cli-cycle` or a cron job. The immune system that only activates after infection is already too late.

---

## II. PHYSICS -- Decay and Isolation

### Token Expiry -> Radioactive Half-Life

Every radioactive isotope has a characteristic half-life -- the time for half the atoms to decay. After ~7 half-lives, less than 1% remains.

| Isotope Analogy | Token Type |
|----------------|------------|
| **Carbon-14** (5,730 years) | API keys with no built-in expiry -- they decay slowly through compromise probability |
| **Iodine-131** (8 days) | OAuth access tokens -- designed for short use |
| **Technetium-99m** (6 hours) | Session tokens, CI/CD ephemeral credentials |
| **Polonium-210** (138 days) | Refresh tokens -- medium-lived, used to generate short-lived access tokens |

**Rule:** Track every credential's half-life. Alert at 1 half-life remaining. For indefinite API keys, define an organizational half-life (e.g., 90 days) and enforce rotation.

### Secret Storage -> Faraday Cage

A Faraday cage is a conductive enclosure that blocks electromagnetic fields. Signals inside cannot leak out; interference outside cannot reach in.

A secret manager (OS keychain, 1Password, Bitwarden, pass) is a Faraday cage for credentials:
- Secrets inside the cage don't appear in config files, logs, git, or process arguments
- External probing (directory listing, file scanning) cannot read the secrets
- Only authorized retrieval through the cage's interface (API call, CLI command) can access them

**Rule:** Every secret should be inside a Faraday cage. Plaintext files on disk are not a cage -- they leak via `cat`, `grep`, backups, and file shares. Env vars are a partial cage -- they don't appear in files but are visible via `/proc/*/environ` and `ps e`.

### Credential Drift -> Entropy

The second law of thermodynamics: in a closed system, entropy (disorder) increases over time. Without energy input, systems degrade.

Credentials are subject to entropy:
- API keys get revoked by admin actions the user doesn't notice
- OAuth tokens expire silently
- Service endpoints change URLs
- Provider deprecates old token formats
- Team member leaves, their tokens are revoked, but referenced in shared config

**Rule:** Credential entropy increases without active management. Doctor mode (periodic health checks) is the energy that fights entropy. A credential audit every 90 days is minimum. Every quarter, run the full scan.

---

## III. Summary Table

| Concept | Analogy | Rule |
|---------|---------|------|
| Token TTL | MHC molecule half-life | Every credential needs an explicit expiry |
| Validation | Thymic positive + negative selection | Two-stage: format check, then API ping |
| Secret isolation | Blood-brain barrier | Config references secrets, never contains them |
| Check ordering | Complement cascade | Check existence -> format -> API, cheapest first |
| Rotation | Arthropod molting (ecdysis) | 4-step: create new, deploy both, verify new, revoke old |
| Periodic audit | Immune surveillance | Continuous monitoring, not just incident response |
| Expiry tracking | Radioactive half-life | Alert at 1 half-life remaining |
| Secret storage | Faraday cage | Secrets inside vault, references outside |
| Credential drift | Thermodynamic entropy | Doctor mode is the energy that fights drift |
