# Containerfile Patterns — Hardened Rootless OCI Images

> **Rule:** Every image produced by this skill MUST follow these patterns.
> A running container is not proof of a secure, operable image.

> **OCI ecosystem:** These patterns work with any OCI-compliant toolchain.
> Adapt commands to your runtime and build tool:
>
> | Purpose | Tools (pick one) |
> |---------|-----------------|
> | Build images | `podman build`, `docker build`, `buildah bud`, `kaniko` |
> | Run containers | `podman run`, `docker run`, `nerdctl run` |
> | Inspect/copy images | `crane`, `skopeo`, `docker inspect` |
> | Declarative units | Quadlet (Podman), Compose (Docker/Podman/nerdctl), systemd units |
> | Image transfer | `skopeo copy`, `crane push/pull`, `podman push` |

---

## 1. Multi-stage build — build, copy, run

Every image uses at least 2 stages. Never ship build tools in the runtime image.

```dockerfile
# ---- Stage 1: build ----
FROM registry.access.redhat.com/ubi9/ubi:latest AS build

RUN dnf install -y --setopt=install_weak_deps=False \
    gcc make openssl-devel \
    && dnf clean all

WORKDIR /build
COPY src/ ./src/
RUN make release

# ---- Stage 2: runtime ----
FROM registry.access.redhat.com/ubi9/ubi-micro:latest AS runtime

# Non-root user (see Section 3)
RUN useradd -r -u 1001 -g 0 -s /usr/sbin/nologin -d /nonexistent svc
USER 1001:0

COPY --from=build --chown=1001:0 /build/bin/app /usr/local/bin/app

ENTRYPOINT ["/usr/local/bin/app"]
```

### Base image selection

| Need | Base image | Why |
|------|-----------|-----|
| Minimal runtime (no shell needed) | `ubi-micro`, `distroless`, `scratch` | Smallest attack surface |
| Runtime needs shell for day-2 ops | `ubi-minimal` (microdnf only) | Allows diagnostics without full dnf |
| Build stage only | `ubi:latest` or language SDK image | Full toolchain, never shipped |
| FIPS required | `ubi9/ubi-minimal` with `crypto-policies-scripts` | See Section 6 |

### Layer caching strategy

```dockerfile
# 1. Base + OS deps (changes rarely)
FROM ubi-micro AS runtime
RUN microdnf install -y ... && microdnf clean all

# 2. App deps (changes sometimes)
COPY --from=build /deps/ /usr/local/lib/

# 3. App binary (changes often)
COPY --from=build /build/bin/app /usr/local/bin/app

# 4. Config templates (changes often)
COPY config/ /etc/app/
```

Order layers from least-changed to most-changed. Never `COPY . .` in a
single layer.

### Image separation per responsibility

| Image | Content | Build pattern |
|-------|---------|---------------|
| **Runtime** | Daemon binary + minimal libs + certs | Multi-stage: build → ubi-micro/distroless |
| **Init** | Schema init, account setup, bootstrap scripts | Multi-stage: build → ubi-minimal (needs shell) |
| **Tools** | Client CLI, export/import, backup/restore, rotation | Multi-stage: build → ubi-minimal |
| **Check** | Health probes, monitoring checks | Multi-stage: build → ubi-micro (single binary preferred) |

---

## 2. Goss / container-structure-test patterns

### Goss template per image type

```yaml
# goss.yaml — runtime image
file:
  /usr/local/bin/app:
    exists: true
    mode: "0755"
    owner: "svc"
  /etc/passwd:
    exists: true
    contains:
      - "svc:x:1001"

user:
  svc:
    exists: true
    uid: 1001
    shell: /usr/sbin/nologin

process:
  app:
    running: true

port:
  tcp:3306:
    listening: true

command:
  "whoami":
    exit-status: 0
    stdout:
      - "svc"
  # No package manager
  "which apk apt dnf yum microdnf rpm dpkg 2>/dev/null":
    exit-status: 1
  # Read-only filesystem check
  "touch /test-write 2>&1":
    exit-status: 1
    stderr:
      - "/test-write: Read-only file system"

http:
  # Use container's own loopback for self-healthcheck
  # For inter-container probes, use DNS name (e.g., http://mariadb:3306)
  http://localhost:8080/healthz:
    status: 200
    timeout: 5000
```

### dgoss workflow

```bash
# Build (podman, docker, or buildah)
podman build -t myapp:test -f Containerfile .

# Test with dgoss (goss wrapper for containers)
dgoss run \
  --read-only \
  --user 1001:0 \
  -v /tmp/test-data:/var/lib/app:Z \
  myapp:test

# Or with container-structure-test
container-structure-test test \
  --image myapp:test \
  --config tests/container-structure.yaml
```

### container-structure-test template

```yaml
schemaVersion: "2.0.0"

fileExistenceTests:
  - name: "binary exists"
    path: "/usr/local/bin/app"
    shouldExist: true
    permissions: "0755"
    uid: 1001

fileContentTests:
  - name: "no root in passwd"
    path: "/etc/passwd"
    expectedContents: ["svc:x:1001"]

commandTests:
  - name: "runs as non-root"
    command: "whoami"
    expectedOutput: ["svc"]
  - name: "no package manager"
    command: "sh"
    args: ["-c", "which apk apt dnf yum 2>/dev/null; echo $?"]
    expectedOutput: ["1"]
  - name: "no shell for svc user"
    command: "getent"
    args: ["passwd", "svc"]
    expectedOutput: ["/usr/sbin/nologin"]

metadataTest:
  labels:
    - key: "org.opencontainers.image.source"
      value: ".*"
      isRegex: true
  user: "1001"
  cmd: []
  healthcheck:
    test: ["CMD", "/usr/local/bin/app", "healthcheck"]
    interval: "30s"
    timeout: "5s"
    retries: 3
```

### What to test per image

| Image | Goss checks |
|-------|-------------|
| **Runtime** | user, binary, ports, process, no pkg manager, no shell, read-only fs, healthcheck |
| **Init** | user, scripts exist, idempotent rerun (run twice → same result), schema version |
| **Tools** | user, CLI help/version, export dry-run, backup dry-run |
| **Check** | user, probe exit codes (0=ok, 1=warn, 2=crit), no mutation, no secrets in output |

---

## 3. USER non-root — mandatory, not optional

**Rule:** Every runtime, tools, and check image MUST run as a non-root user.
"When possible" is not acceptable. If the service requires root, the architecture
is wrong — fix the architecture.

```dockerfile
# Create user in build stage or early layer
RUN useradd -r -u 1001 -g 0 -s /usr/sbin/nologin -M -d /nonexistent svc

# Set ownership on required paths only
RUN install -d -o 1001 -g 0 -m 0750 /var/lib/app /var/log/app /tmp/app

# Switch to non-root BEFORE any COPY of app files
USER 1001:0

COPY --from=build --chown=1001:0 /build/bin/app /usr/local/bin/app
```

### UID/GID conventions

| Account | UID | Purpose |
|---------|-----|---------|
| `svc` | 1001 | Runtime daemon |
| `tools` | 1002 | Tools/backup (if separate) |
| `check` | 1003 | Monitoring probes (if separate) |

Group 0 (root group) is intentional — OpenShift compatibility. The user has no
root privileges; group 0 membership only enables file access in constrained
environments.

### Init image exception

The init image MAY temporarily run as root for schema/account bootstrap, but
MUST drop privileges before completing:

```dockerfile
# Init runs as root for bootstrap
USER 0
COPY --chown=0:0 init.sh /usr/local/bin/init.sh
# But the init script itself must:
#   1. Do privileged work (chown dirs, etc.)
#   2. exec gosu svc <app> OR just exit (Quadlet starts runtime as svc)
```

---

## 4. HEALTHCHECK — mandatory in every runtime image

```dockerfile
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD ["/usr/local/bin/app", "healthcheck"]
```

### Quadlet Health directives

```ini
# app.container (Quadlet)
[Container]
Image=localhost/myapp:latest
User=1001
ReadOnly=true
HealthCmd=/usr/local/bin/app healthcheck
HealthInterval=30s
HealthTimeout=5s
HealthRetries=3
HealthStartPeriod=10s
```

### Healthcheck design rules

| Rule | Rationale |
|------|-----------|
| Use a compiled binary, not `curl` or `wget` | Don't add tools to minimal images |
| Check the actual service protocol, not just PID | `pidof` is not a health check |
| Exit 0 = healthy, Exit 1 = unhealthy | OCI standard |
| No secrets in healthcheck commands | Visible in `podman/docker inspect` |
| Healthcheck runs as the same USER | No privilege escalation |
| Start period covers real init time | Avoid false-negative during schema migration |

---

## 5. CVE scan and SBOM — mandatory T1 gate

**Rule:** Every image MUST pass a vulnerability scan before deployment.
This is a T1 blocking gate, not a XL-only option.

The skill is tool-agnostic. Use whichever scanner is available in the project's
toolchain. Common options:

| Tool | CVE scan | SBOM | Notes |
|------|----------|------|-------|
| Trivy | `trivy image --severity HIGH,CRITICAL --exit-code 1` | `trivy image --format spdx-json` | Most common, Aqua Security |
| Grype | `grype --fail-on high` | Use syft for SBOM | Anchore |
| Snyk | `snyk container test` | `snyk container sbom` | Commercial, free tier |
| Clair | Quay integrated | Separate | RedHat/Quay native |
| Docker Scout | `docker scout cves` | `docker scout sbom` | Docker Desktop |

### Scan workflow (examples)

```bash
# Example A: Trivy
trivy image --severity HIGH,CRITICAL --exit-code 1 myapp:latest
trivy image --format spdx-json -o sbom.spdx.json myapp:latest

# Example B: Grype + Syft
grype myapp:latest --fail-on high
syft myapp:latest -o spdx-json > sbom.spdx.json

# Example C: Snyk
snyk container test myapp:latest --severity-threshold=high
```

### CVE threshold policy

| Severity | Policy |
|----------|--------|
| CRITICAL | Block: fix or document accepted risk with owner + expiry date |
| HIGH | Block: fix or document accepted risk |
| MEDIUM | Warn: track in backlog |
| LOW | Info: ignore unless cumulative |

### CI integration

```makefile
# In Makefile or CI — adapt scanner command to your tool
scan: build
	$(SCANNER) $(IMAGE):$(TAG)  # must exit non-zero on HIGH/CRITICAL
	$(SBOM_TOOL) $(IMAGE):$(TAG) -o sbom.spdx.json

# Gate: scan MUST pass before push to registry
push: scan
	podman push $(IMAGE):$(TAG)
```

### Image signing (air-gapped / regulated)

```bash
# Sign with cosign (sigstore)
cosign sign --key cosign.key $(REGISTRY)/$(IMAGE):$(TAG)

# Verify before deploy
cosign verify --key cosign.pub $(REGISTRY)/$(IMAGE):$(TAG)
```

---

## 6. FIPS mode — federal/regulated compliance

### Enable FIPS in the final image

```dockerfile
FROM registry.access.redhat.com/ubi9/ubi-minimal:latest AS runtime

# Install FIPS crypto policy
RUN microdnf install -y crypto-policies-scripts \
    && update-crypto-policies --set FIPS \
    && microdnf remove -y crypto-policies-scripts \
    && microdnf clean all

# Verify
RUN update-crypto-policies --show | grep -q FIPS
```

### FIPS validation in goss

```yaml
command:
  "update-crypto-policies --show":
    exit-status: 0
    stdout:
      - "FIPS"
  # OpenSSL must use FIPS provider
  "openssl list -providers 2>/dev/null | grep -i fips || echo no-fips":
    exit-status: 0
    stdout:
      - /fips|FIPS/
```

### FIPS constraints

| Constraint | Impact |
|------------|--------|
| MD5 disabled | No MD5 checksums, password hashes, or digests |
| SHA-1 disabled for signatures | May break old TLS certs |
| Minimum TLS 1.2 | No TLS 1.0/1.1 |
| RSA minimum 2048 bits | Short keys rejected |
| Certain ciphers unavailable | Application must support FIPS-approved ciphers |
| Performance impact | ~5-15% on crypto operations |

### Host FIPS propagation

If the host runs in FIPS mode, the container inherits it via the kernel. But
the image's crypto libraries must also be FIPS-aware. Always validate both:

```bash
# Host FIPS check
cat /proc/sys/crypto/fips_enabled  # 1 = FIPS

# Container FIPS check (must also show FIPS)
podman run --rm myapp:latest update-crypto-policies --show
```

---

## 7. Strip everything — remove package managers, shells, and unnecessary tools

**Rule:** The final runtime image must contain ONLY what the service needs to run.
Everything else is attack surface.

### What to remove

```dockerfile
# After installing runtime deps, remove the package manager itself
RUN microdnf install -y required-lib-only \
    && microdnf clean all \
    && rpm -e --nodeps microdnf rpm \
    || true

# Remove shells if the service doesn't need one
RUN rm -f /bin/sh /bin/bash /usr/bin/sh /usr/bin/bash 2>/dev/null || true

# Remove unnecessary system binaries
RUN rm -rf \
    /usr/bin/curl /usr/bin/wget \
    /usr/bin/vi /usr/bin/vim \
    /usr/bin/find /usr/bin/locate \
    /usr/bin/nc /usr/bin/ncat \
    /usr/bin/ss /usr/bin/ip \
    /usr/sbin/useradd /usr/sbin/usermod \
    2>/dev/null || true

# Remove docs, man pages, locales
RUN rm -rf /usr/share/doc /usr/share/man /usr/share/info \
    /usr/share/locale /usr/share/i18n \
    /var/cache/* /var/log/* /tmp/*
```

### Preferred approach: distroless or scratch

Instead of removing tools from a full image, start from nothing:

```dockerfile
# Best: scratch (for static binaries)
FROM scratch AS runtime
COPY --from=build /build/bin/app /app
COPY --from=build /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
USER 1001:0
ENTRYPOINT ["/app"]

# Good: distroless (for dynamic binaries needing glibc)
FROM gcr.io/distroless/base-nossl-debian12:nonroot AS runtime
COPY --from=build /build/bin/app /app
ENTRYPOINT ["/app"]
```

### Verify nothing unnecessary remains

```yaml
# goss.yaml — verify stripped image
command:
  # No package manager
  "which apk apt apt-get dnf yum microdnf rpm dpkg pip pip3 gem npm 2>/dev/null; echo $?":
    exit-status: 0
    stdout:
      - "1"
  # No interactive shell for service user
  "getent passwd svc":
    exit-status: 0
    stdout:
      - /nologin|false/
  # No compiler or build tools
  "which gcc cc make cmake g++ 2>/dev/null; echo $?":
    exit-status: 0
    stdout:
      - "1"
  # No network debug tools
  "which curl wget nc ncat telnet nmap ss 2>/dev/null; echo $?":
    exit-status: 0
    stdout:
      - "1"
```

---

## 8. Read-only filesystem — mandatory for rootless runtime

### OCI runtime (podman/docker/nerdctl)

```bash
# Works with podman, docker, or nerdctl — same flags
podman run \
  --read-only \
  --tmpfs /tmp:rw,noexec,nosuid,size=64m \
  --tmpfs /run:rw,noexec,nosuid,size=64m \
  -v app-data:/var/lib/app:Z \
  myapp:latest
```

### Quadlet unit (Podman) / Compose equivalent

```ini
[Container]
Image=localhost/myapp:latest
User=1001
ReadOnly=true
Tmpfs=/tmp:rw,noexec,nosuid,size=64m
Tmpfs=/run:rw,noexec,nosuid,size=64m
Volume=app-data.volume:/var/lib/app:Z
```

### What needs tmpfs

| Path | Why | Flags |
|------|-----|-------|
| `/tmp` | Temp files, sockets | `rw,noexec,nosuid,size=64m` |
| `/run` | PID files, runtime sockets | `rw,noexec,nosuid,size=64m` |
| `/var/log/app` | Logs (if not stdout) | `rw,noexec,nosuid` — or use a named volume |

### What needs a named volume

| Path | Why |
|------|-----|
| `/var/lib/app` | Persistent data (DB, state) |
| `/etc/app/certs` | TLS certs mounted from host |
| `/var/backup` | Backup archives |

### Verify read-only in goss

```yaml
command:
  "touch /test-readonly 2>&1":
    exit-status: 1
    stderr:
      - "Read-only file system"
  "touch /usr/test-readonly 2>&1":
    exit-status: 1
  # tmpfs is writable
  "touch /tmp/test-writable":
    exit-status: 0
  # Data volume is writable
  "touch /var/lib/app/test-writable":
    exit-status: 0
```

---

## 9. Inter-container communication — never rely on localhost alone

**Rule:** When multiple containers must communicate (DB ↔ App ↔ Cache ↔ Monitoring),
NEVER assume `127.0.0.1`. Use explicit, resilient communication paths.

### Decision matrix

| Method | When to use | Resilience | Rootless support |
|--------|-------------|------------|-----------------|
| **Podman pod** (shared localhost) | Tightly coupled services that MUST co-schedule | Low — no isolation, all-or-nothing restart | Full |
| **User-defined network + DNS** | Independent services needing discovery by name | High — restart-tolerant, name-based | Full (netavark/CNI) |
| **Unix socket (bind mount)** | Same-host, high-throughput, zero-network (DB ↔ App) | High — no port, no TCP overhead | Full |
| **Systemd socket activation** | Lazy-start, on-demand service wakeup | High — native supervision | Quadlet only |
| **Host network (--network=host)** | Legacy compat, monitoring bridges | Low — no isolation | Full but defeats purpose |

### Recommended: user-defined network + container DNS

```bash
# Create a rootless user network (persists across reboots via Quadlet)
podman network create app-internal

# Containers resolve each other by name
podman run -d --network app-internal --name mariadb myapp-mariadb:latest
podman run -d --network app-internal --name central myapp-central:latest
# central can reach mariadb at: mariadb:3306 (DNS name, NOT localhost)
```

#### Quadlet equivalent

```ini
# app-internal.network (Quadlet)
[Network]
Driver=bridge
Internal=true
DNS=true

# mariadb.container (Quadlet)
[Container]
Image=localhost/myapp-mariadb:latest
Network=app-internal.network
User=1001
ReadOnly=true

# central.container (Quadlet)
[Container]
Image=localhost/myapp-central:latest
Network=app-internal.network
User=1001
ReadOnly=true
Environment=DB_HOST=mariadb
Environment=DB_PORT=3306
```

**Key points:**
- `Internal=true` → no external access, containers only reach each other
- `DNS=true` → containers resolve by name (not IP)
- No port mapping needed between containers on the same network
- Only expose ports to the host when external access is required

#### Compose equivalent

```yaml
services:
  mariadb:
    image: myapp-mariadb:latest
    networks: [internal]
    read_only: true
    user: "1001:0"

  central:
    image: myapp-central:latest
    networks: [internal]
    read_only: true
    user: "1001:0"
    environment:
      DB_HOST: mariadb
      DB_PORT: "3306"
    depends_on:
      mariadb:
        condition: service_healthy

networks:
  internal:
    internal: true  # no external access
```

### Unix socket pattern (DB ↔ App, highest performance)

```bash
# Host creates shared socket directory
install -d -o 1001 -g 0 -m 0750 /run/app/sockets

# MariaDB exposes socket, not TCP
podman run -d \
  --name mariadb \
  -v /run/app/sockets:/var/run/mysqld:Z \
  myapp-mariadb:latest

# App connects via socket, not TCP
podman run -d \
  --name central \
  -v /run/app/sockets:/var/run/mysqld:ro,Z \
  -e DB_SOCKET=/var/run/mysqld/mysqld.sock \
  myapp-central:latest
```

**Advantages over TCP localhost:**
- No TCP overhead (30-40% faster for local queries)
- No port conflict risk
- No port exposure to host
- File permissions = access control (no firewall needed)

#### Quadlet with socket

```ini
# mariadb.container
[Container]
Volume=/run/app/sockets:/var/run/mysqld:Z

# central.container
[Container]
Volume=/run/app/sockets:/var/run/mysqld:ro,Z
Environment=DB_SOCKET=/var/run/mysqld/mysqld.sock
```

### Anti-patterns

| Anti-pattern | Problem | Fix |
|---|---|---|
| `--network=host` everywhere | No isolation, port conflicts, defeats rootless | Use user-defined network |
| Hardcoded `127.0.0.1:3306` | Breaks when containers are on different networks or pods | Use DNS name or socket |
| Hardcoded container IP | IPs change on restart | Use DNS names |
| `localhost` in config files | Ambiguous: host's localhost or container's? | Use explicit service name or socket path |
| No `Internal=true` on app network | Services accidentally reachable from host/outside | Always `Internal=true` for backend networks |
| Shared pod for unrelated services | One crash restarts everything, no independent scaling | Pod only for tightly coupled (sidecar pattern) |

### When to use a pod (shared localhost)

Pods are appropriate ONLY for the sidecar pattern — tightly coupled containers
that MUST share network namespace:

- App + log collector (fluentbit)
- App + TLS proxy (envoy/traefik sidecar)
- App + metrics exporter (prometheus exporter)

```bash
podman pod create --name app-pod -p 8080:8080
podman run -d --pod app-pod --name app myapp:latest
podman run -d --pod app-pod --name metrics myapp-exporter:latest
# metrics can reach app at localhost:8080 (same network namespace)
```

For everything else (DB, cache, message broker, monitoring), use a **user-defined network**.

### Goss validation for inter-container communication

```yaml
# goss.yaml — verify DNS resolution works
command:
  # Can resolve peer by name (not IP)
  "getent hosts mariadb":
    exit-status: 0
  # Connection works
  "timeout 5 bash -c 'echo > /dev/tcp/mariadb/3306'":
    exit-status: 0

# goss.yaml — verify socket communication works
file:
  /var/run/mysqld/mysqld.sock:
    exists: true
    filetype: socket
```

---

## 10. Checklist — mandatory for every image

Before declaring T1 gate passed, verify ALL items:

```
[ ] Multi-stage build: no build tools in runtime image
[ ] USER non-root: runs as UID 1001+ with /usr/sbin/nologin
[ ] HEALTHCHECK: present, uses app binary, no curl/wget
[ ] Read-only filesystem: --read-only + tmpfs for /tmp, /run
[ ] No package manager: apk/apt/dnf/yum/rpm removed or absent
[ ] No shell (if possible): /bin/sh removed or service user has /nologin
[ ] No build tools: gcc/make/cmake absent
[ ] No network debug: curl/wget/nc/ncat/ss absent
[ ] FIPS: crypto-policies set to FIPS (if regulated)
[ ] CVE scan: trivy/grype HIGH+CRITICAL = 0 or documented exceptions
[ ] SBOM: spdx-json generated and archived
[ ] Goss/CST: tests pass for user, files, ports, process, stripped tools
[ ] Image signed (if registry requires): signature verification passes (cosign, notation, or registry-native)
[ ] Labels: org.opencontainers.image.* metadata present
[ ] Layer order: least-changed → most-changed for cache efficiency
```
