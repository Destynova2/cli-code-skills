# HLD Section Templates

> **When to read:** During Step 5 of the workflow — when writing the HLD. Only include sections selected by the mitosis matrix in Step 3.

---

### Section 1 — Executive Summary

2-3 paragraphs max. Cover:
- What problem does this system solve?
- What is the proposed approach?
- What are the key tradeoffs?

**Write this LAST**, after completing all other sections.

### Section 2 — Goals and Non-Goals

**Source:** Google Design Docs tradition

```markdown
**Goals:**
- G1: [Specific, measurable goal]
- G2: ...

**Non-Goals (explicitly out of scope):**
- NG1: [What this design intentionally does NOT address, and why]
- NG2: ...
```

Non-goals prevent scope creep. Every "non-goal" should be something that *could reasonably be a goal* but is explicitly excluded.

### Section 3 — Stakeholders & Constraints

**Source:** arc42 §1-2, French DAT model

**Stakeholders:**

| Stakeholder | Role | Key Concern |
|-------------|------|-------------|
| End users | Use the product | Latency, UX |
| Ops team | Run the system | Observability, deployment ease |
| Security team | Audit compliance | ANSSI/RGPD conformity |
| Product owner | Define features | Time-to-market |

**Constraints:**

| Type | Constraint | Impact |
|------|-----------|--------|
| Technical | Must run on Kubernetes 1.28+ | Limits deployment options |
| Regulatory | RGPD / data sovereignty | Data must stay in EU |
| Organizational | Team of 4 devs | Limits complexity |
| Budget | AWS budget < $5K/month | Limits scaling options |

### Section 4 — System Context (C4 Level 1)

**Source:** Simon Brown, C4 Model

```mermaid
graph TB
    User[👤 User] -->|HTTPS| System[🖥️ System Name]
    Admin[👤 Admin] -->|HTTPS| System
    System -->|REST| ExtAPI[📡 External API]
    System -->|SMTP| Mail[📧 Email Service]
    System -->|SQL| DB[(🗄️ Database)]
    Monitor[📊 Monitoring] -.->|scrape| System
```

Describe:
- Who uses the system (actors)
- What external systems it integrates with
- What the system boundary is (in scope vs out)
- Communication protocols on every arrow

### Section 5 — Solution Strategy

**Source:** arc42 §4

High-level approach in 3-5 bullet points:
- Architecture style choice (monolith, modular monolith, microservices, serverless) with rationale
- Key technology decisions (language, framework, database type)
- Fundamental patterns (event-driven, CQRS, request-response, batch)
- Integration strategy (sync REST, async messaging, hybrid)

This section is the "elevator pitch" of the architecture.

### Section 6 — Container Architecture (C4 Level 2)

**Source:** Simon Brown, C4 Model

```mermaid
graph TB
    subgraph "System Boundary"
        FE[🌐 Frontend<br/>React/Next.js]
        API[⚙️ API Gateway<br/>Kong/Envoy]
        SVC1[📦 Service A<br/>Rust/Go]
        SVC2[📦 Service B<br/>Python]
        MQ[📬 Message Queue<br/>NATS/Kafka]
        DB[(🗄️ Database<br/>PostgreSQL)]
        Cache[(⚡ Cache<br/>Redis)]
    end

    User[👤 User] --> FE
    FE --> API
    API --> SVC1
    API --> SVC2
    SVC1 --> DB
    SVC1 --> Cache
    SVC1 --> MQ
    MQ --> SVC2
```

For each container:

| Container | Technology | Responsibility | Scaling Strategy |
|-----------|-----------|---------------|-----------------|
| API Gateway | Kong | Routing, rate limiting, auth | Horizontal, stateless |
| Service A | Rust | Core business logic | Horizontal behind LB |
| Database | PostgreSQL | Persistent storage | Primary-replica |
| Cache | Redis | Hot data, sessions | Redis Cluster |

**Label every arrow** with protocol (HTTPS, gRPC, AMQP, SQL) and data format (JSON, Protobuf).

### Section 7 — Capacity Estimations

Present results from Step 4:

| Metric | Value | Formula / Assumption |
|--------|-------|---------------------|
| DAU | X | Given/estimated |
| Avg QPS | X | DAU × req/user/day ÷ 86400 |
| Peak QPS | X | Avg QPS × 3 |
| Storage/year | X TB | objects/day × avg size × 365 |
| Bandwidth | X Gbps | Peak QPS × avg response size |
| Cache size | X GB | 20% × total hot data |
| Min servers | X | Peak QPS ÷ QPS/server × 1.3 |

**Scaling decision points:**

| Load Level | Strategy |
|-----------|----------|
| < 1K QPS | Single server, vertical scaling |
| 1K-10K QPS | Read replicas, caching |
| 10K-100K QPS | Horizontal scaling, sharding, CDN |
| 100K+ QPS | Microservices, multi-region, event-driven |

### Section 8 — Domain Boundaries (Strategic DDD)

**Source:** Eric Evans (DDD), Vaughn Vernon, context mapping

> This section is optional for simple systems. It becomes essential for multi-team / multi-service systems.

**Bounded contexts:**

Identify the major domain boundaries in the system:

```mermaid
graph TB
    subgraph "Order Context"
        O[Order]
        OL[OrderLine]
    end
    subgraph "Inventory Context"
        S[Stock]
        W[Warehouse]
    end
    subgraph "Payment Context"
        P[Payment]
        R[Refund]
    end

    O -.->|"Domain Event: OrderCreated"| S
    O -.->|"Domain Event: OrderCreated"| P
    P -.->|"Domain Event: PaymentCompleted"| O
```

**Context map (relationships between bounded contexts):**

| Upstream | Downstream | Relationship | Integration |
|----------|-----------|-------------|-------------|
| Order | Inventory | Customer-Supplier | Domain events via MQ |
| Payment (external) | Order | Conformist | REST API, adapt to their model |
| Order | Notification | Published Language | Domain events |

**Relationship types:** Shared Kernel, Customer-Supplier, Conformist, Anti-Corruption Layer, Open Host Service, Published Language, Separate Ways.

Each bounded context typically maps to one container in the C4 L2 diagram. If two contexts are in the same container, document why (e.g., too small to justify a separate service).

> **Note:** Tactical DDD (aggregates, entities, value objects, invariants) belongs in the LLD for each component.

### Section 9 — Data Architecture

Cover:
- **Data model** (high-level ER diagram — entities and relationships only, no column detail)
- **Storage strategy**: which data in SQL vs NoSQL vs object store vs cache, with rationale
- **Data flow**: write path and read path through the system
- **Partitioning/sharding** strategy (if needed based on capacity estimation)
- **Retention and lifecycle**: how long is data kept, archival strategy
- **Backup and recovery**: RPO/RTO targets

```mermaid
erDiagram
    USER ||--o{ ORDER : places
    ORDER ||--|{ LINE_ITEM : contains
    PRODUCT ||--o{ LINE_ITEM : "is in"
```

### Section 10 — API Design (High-Level)

List the main API surfaces (not full specs — that's LLD territory):

| Endpoint | Method | Purpose | Auth |
|----------|--------|---------|------|
| /api/v1/users | GET | List users | JWT |
| /api/v1/orders | POST | Create order | JWT |
| /api/v1/health | GET | Health check | None |

Specify:
- Versioning strategy (URL path `/v1/`, header, or content negotiation)
- Rate limiting policy (global, per-user, per-tenant)
- Pagination convention (cursor-based recommended)
- Error format convention (RFC 7807 problem details recommended)

### Section 11 — Security Architecture & Trust Boundaries

**Source:** ANSSI, OWASP, ISO 27001, Microsoft STRIDE (system level)

**Security policies:**

| Aspect | Approach |
|--------|----------|
| Authentication | JWT via OIDC provider / mTLS between services |
| Authorization | RBAC/ABAC with policy engine (OPA, Cedar) |
| Encryption at rest | AES-256 via provider KMS |
| Encryption in transit | TLS 1.3, mTLS for service-to-service |
| Network security | Zero-trust, network policies, no public endpoints |
| Secrets management | Vault/OpenBao, automatic rotation |
| Supply chain | Image signing, SBOM, vulnerability scanning |
| Compliance | [ANSSI / RGPD / PCI-DSS / relevant standard] |

**Trust boundaries (STRIDE at system level):**

Identify trust boundaries on the C4 L2 container diagram:

```
Internet ──[TLS]──► API Gateway ──[mTLS]──► Internal Services ──[TLS]──► Database
   ▲                    ▲                         ▲
   │                    │                         │
 Untrusted          DMZ / edge               Trusted zone
```

| Boundary | STRIDE threats at this level | Mitigation |
|----------|----------------------------|-----------|
| Internet → API Gateway | Spoofing (forged identity), DoS | JWT validation, rate limiting, WAF |
| API Gateway → Services | Tampering (modified requests), Elevation | mTLS, input validation, RBAC |
| Services → Database | Info Disclosure, Tampering | Encryption at rest, network isolation, least-privilege DB users |
| Services → External APIs | Info Disclosure (credential leak) | Secrets in vault, no keys in logs/errors |

> **Note:** Component-level STRIDE (SQL injection in repository, IDOR in controller) belongs in the LLD. HLD covers system-level trust boundaries only.

### Section 12 — Deployment Architecture

```mermaid
graph TB
    subgraph "Production"
        subgraph "Zone A"
            N1[Node 1]
            N2[Node 2]
        end
        subgraph "Zone B"
            N3[Node 3]
            N4[Node 4]
        end
        LB[Load Balancer] --> N1 & N2 & N3 & N4
    end
    subgraph "CI/CD"
        Git[Git] --> Pipeline[CI Pipeline]
        Pipeline --> Registry[Container Registry]
        Registry --> |GitOps| Production
    end
```

Cover:
- **Infrastructure**: bare metal, VMs, K8s, cloud provider
- **IaC/GitOps**: Terraform, ArgoCD, FluxCD
- **CI/CD pipeline**: stages, quality gates, rollback strategy
- **Environments**: dev → staging → prod, environment parity
- **Scaling**: horizontal/vertical, autoscaling triggers

### Section 13 — Observability & SLOs

**The three pillars:**
- **Metrics**: what's collected, where (Prometheus, VictoriaMetrics), key dashboards
- **Logs**: centralized (Loki, ELK), structured format, retention
- **Traces**: distributed tracing (Jaeger, Tempo), critical paths instrumented

**SLIs and SLOs:**

| SLI | SLO | Error Budget |
|-----|-----|-------------|
| Request latency p99 | < 500ms | 0.1% can exceed |
| Availability | 99.9% | 8.76h downtime/year |
| Error rate | < 0.1% | — |

### Section 14 — Failure Modes and Mitigation

| Failure Mode | Impact | Probability | Mitigation | Detection |
|-------------|--------|-------------|-----------|-----------|
| DB primary down | Write unavailable | Low | Auto-failover to replica | Health check + alert |
| Cache failure | Increased latency | Medium | Fallback to DB, circuit breaker | Latency spike alert |
| Network partition | Partial outage | Low | Multi-AZ, retry with backoff | Connectivity monitoring |
| Provider API down | Feature degraded | Medium | Circuit breaker + fallback | Error rate alert |

### Section 15 — Architecture Decision Records (ADRs)

**Source:** MADR (Markdown Any Decision Record), Michael Nygard

For each major decision:

```markdown
### ADR-NNN: [Title]

**Status:** Proposed | Accepted | Deprecated | Superseded by ADR-XXX

**Context:**
What is the problem or situation that requires a decision?

**Decision Drivers:**
- [driver 1]
- [driver 2]

**Considered Options:**

| Option | Pros | Cons |
|--------|------|------|
| Option A (chosen) | ... | ... |
| Option B | ... | ... |
| Option C | ... | ... |

**Decision:** [Chosen option and why]

**Consequences:**
- Good: ...
- Bad: ...
- Risks: ...

**Quality Attributes Affected:** Performance | Security | Scalability | Maintainability | Cost
```

**Typical ADRs in an HLD:**
- ADR-001: Architecture style (monolith vs microservices vs modular monolith)
- ADR-002: Primary database engine
- ADR-003: Communication pattern (sync REST vs async messaging)
- ADR-004: Deployment platform
- ADR-005: Authentication mechanism

### Section 16 — Tradeoff Analysis

**Source:** ATAM (Architecture Tradeoff Analysis Method), Bass/Clements/Kazman

**Quality Attribute Utility Tree:**

```
Utility
├── Performance
│   ├── [H,H] API response < 200ms p95 under normal load
│   └── [M,H] Batch processing < 1h for 1M records
├── Availability
│   ├── [H,H] 99.9% uptime
│   └── [M,M] Graceful degradation under partial failure
├── Security
│   ├── [H,H] No credential leakage in responses/logs
│   └── [H,M] All data encrypted at rest
└── Scalability
    ├── [M,H] Handle 10x current load within 15min
    └── [L,M] Multi-region deployment ready
```

Format: `[Business importance, Technical difficulty]` — H/M/L

**Sensitivity points** (one decision affects one attribute):
- "Database choice affects query performance"

**Tradeoff points** (one decision affects multiple attributes in opposite directions):
- "Microservices improve scalability but reduce operational simplicity"

**Weighted decision matrix** (for key technology choices):

| Criterion | Weight | Option A | Score | Option B | Score |
|-----------|--------|----------|-------|----------|-------|
| Performance | 5 | Good (4) | 20 | Excellent (5) | 25 |
| Operational cost | 3 | Low (5) | 15 | High (2) | 6 |
| Team familiarity | 4 | High (5) | 20 | Low (2) | 8 |
| **Total** | | | **55** | | **39** |

### Section 17 — Risks and Open Questions

| # | Risk/Question | Owner | Deadline | Status | Mitigation |
|---|--------------|-------|----------|--------|-----------|
| R1 | Data migration from legacy | @architect | Sprint 3 | Open | Dual-write phase |
| R2 | Performance under load not validated | @lead | Before Gate | Open | Load test in staging |
| Q1 | Which OIDC provider? | @security | Week 2 | Open | — |

### Section 18 — Glossary

**Source:** DDD ubiquitous language, arc42 §12

| Term | Definition |
|------|-----------|
| [Domain term] | [What it means in this system's context] |
