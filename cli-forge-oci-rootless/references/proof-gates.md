# Proof Gates — Lithic OCI Rootless Migration

The migration verdict must be based on executable proof, not architectural
assertion. Use this matrix to turn claims into gates.

## Gate notation

Each gate should record:

```markdown
| Gate | Capability | Command or method | Evidence artifact | Automatable? | Owner | Blocking? |
```

- **Blocking** means convergence cannot be claimed without it.
- Manual gates are acceptable only when named, repeatable, and paired with an
automation target.
- For rootless services, clean deploy, dirty rerun, restart, and reboot are
different proofs.

## T0 — Bedrock / contract gates

Purpose: prove the operational contract is explicit and internally consistent.

Recommended gates:

- Parse all YAML/JSON/TOML/env/Quadlet/systemd inputs.
- Validate instance manifest schema if present.
- Check ports are declared once and reflected in docs/CLI/Quadlet.
- Check service/operator/monitoring/backup accounts are declared once.
- Check data/log/backup/cert paths are declared once.
- Check secret names exist in the manifest without exposing secret values.
- Check public CLI verbs cover start/stop/status/logs/client/backup/restore/check/diagnose where required.
- Check docs do not mention commands absent from CLI or wrappers.
- Check compatibility wrappers call the public CLI or are marked tailings.
- Check generated Quadlet manifests are reproducible from the source of truth.

Minimum blocking gates for stateful migration:

- contract inventory exists
- storage contract exists
- day-2 command inventory exists
- backup/restore contract exists
- monitoring local/remote contract exists if monitoring is required

## T1 — Alloy / component gates

Purpose: prove the refined artifacts build and are individually sane.

Recommended gates:

- Build runtime image (multi-stage, no build tools in final layer).
- Build tools image.
- Build check image.
- Build init image or validate init logic.
- Smoke runtime image without host secrets.
- Smoke tools image help/version.
- Smoke check image help and example failure.
- Verify image labels/metadata (org.opencontainers.image.*).
- Verify image runs as non-root (UID 1001+, /usr/sbin/nologin).
- Verify read-only filesystem works (--read-only + tmpfs /tmp, /run).
- Verify no package manager present (apk/apt/dnf/yum/rpm/pip removed).
- Verify no shell for service user (or no shell at all in distroless/scratch).
- Verify no network debug tools (curl/wget/nc/ncat/ss absent).
- Verify HEALTHCHECK directive present and uses app binary (not curl).
- Run goss/container-structure-test with hardening checks.
- Run CVE scan (vulnerability scanner (trivy, grype, or equivalent)): HIGH+CRITICAL = 0 or documented exception.
- Generate SBOM (spdx-json) and archive with build.
- Verify FIPS crypto-policies if regulated environment.
- Ensure no production secrets are baked into images.

Minimum blocking gates:

- runtime image build (multi-stage)
- non-root USER (UID 1001+)
- read-only filesystem
- no package manager in runtime image
- HEALTHCHECK present
- CVE scan pass (HIGH+CRITICAL = 0 or documented)
- SBOM generated
- goss/container-structure-test pass
- tools/check availability for required day-2 and monitoring
- no secret bake-in

## T2 — Fresh formation / deploy gates

Purpose: prove a clean host can become a running rootless instance.

Recommended gates:

- Create or validate service account.
- Validate subuid/subgid.
- Validate rootless Podman availability.
- Validate `systemd --user` availability for service account.
- Enable lingering if required and documented.
- Create host directories with expected ownership and labels.
- Render secrets/certs placeholders or test secrets with correct permissions.
- Install/generate Quadlet files in correct user scope.
- Run daemon-reload for the user manager.
- Start service via `systemctl --user`.
- Check status via public CLI.
- Fetch logs via public CLI.
- Restart via public CLI.
- Re-run bootstrap idempotently.
- Reboot host or test VM and confirm service state if production requires it.
- Remove/cleanup instance cleanly if uninstall is in scope.

Minimum blocking gates:

- clean deploy
- rootless start through `systemd --user`
- status/logs through CLI
- idempotent rerun
- reboot survival if required in production

## T3 — Field operations / day-2 gates

Purpose: prove operators can run the product, not merely start it.

Recommended gates:

- Local client access.
- External client/protocol access.
- SQL or application protocol smoke if relevant.
- Local check succeeds.
- Remote monitoring path succeeds.
- Monitoring failure has expected exit code/message.
- Export works.
- Import works.
- Backup creates archive.
- Backup verification is independent from creation.
- Restore works on the same host.
- Restore works on a clean host or isolated test instance.
- WAL/binlog/journal replay works if required.
- Purge/retention works without deleting live state.
- Account rotation works.
- Secret/cert rotation reconciles running service state.
- Diagnostics bundle or command collects expected facts without secrets.
- Compatibility wrappers preserve documented semantics.

Minimum blocking gates:

- day-2 CLI commands for critical operations
- local and remote monitoring when required
- backup + verify + restore
- compatibility semantics if wrappers remain

## T4 — Stress & fracture gates

Purpose: prove known cracks are detected safely.

Recommended injections:

- Missing secret.
- Invalid secret.
- Missing certificate.
- Expired certificate if testable.
- Wrong port or port already in use.
- Broken config syntax.
- Missing image.
- Registry unavailable.
- User bus unavailable.
- Lingering disabled when required.
- Incorrect UID/GID mapping.
- SELinux/AppArmor denial or wrong label.
- Data directory unwritable.
- Disk full or low space.
- Backup archive missing.
- Backup archive corrupt.
- Clock skew if TLS or replication is sensitive.
- Remote monitoring bridge broken while local check is green.
- Stale root systemd unit still present.
- Legacy wrapper tries to mutate state directly.

Expected result:

- failure is detected
- message is actionable
- exit code is stable where automation depends on it
- service does not corrupt state
- runbook tells operator what to do

Minimum blocking gates:

- missing/invalid secret behavior
- broken config behavior
- rootless/systemd failure behavior
- restore failure behavior
- remote monitoring failure behavior if remote monitoring is required

## M0 — Stratigraphic memory gates

Purpose: turn surprises into durable layers of knowledge.

For every incident, failed test, or migration surprise, create:

- runbook delta
- blackbox or incident entry
- anti-regression test
- clarified bedrock invariant
- owner and due date for unresolved debt
- tailings entry if legacy residue remains

Minimum blocking gates:

- migration report has residual debt section
- each accepted caveat has owner/condition
- each critical manual proof has automation target
