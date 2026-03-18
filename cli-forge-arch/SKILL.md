---
name: cli-forge-arch
description: "Use this skill whenever the user wants to create, review, or iterate on a High-Level Design (HLD) or Low-Level Design (LLD) document for a software system. Triggers include: any mention of 'HLD', 'LLD', 'system design', 'architecture document', 'design document', 'technical design', 'design doc', 'ADR', 'architecture decision record', 'C4 diagram', 'back-of-envelope', 'capacity planning', 'ATAM', or requests to design/architect a system, service, platform, or infrastructure. Also triggers when the user asks to evaluate architecture tradeoffs, estimate system capacity, write a design proposal, create a technical specification, or produce a Gate review document. Use this skill even if the user just says 'design X' or 'architect Y' — any system-level design work should use this skill. Do NOT use for UI/UX design, graphic design, or database schema design in isolation (without broader system context)."
argument-hint: "[system-name-or-description]"
context: fork
agent: general-purpose
allowed-tools:
  - Read
  - Write
  - Grep
  - Glob
  - Bash
  - WebSearch
  - WebFetch
---

# Arch Forge — System Design (HLD & LLD)

Produce structured, professional High-Level Design (HLD) and Low-Level Design (LLD) documents following industry frameworks (C4 Model, ATAM, ADR) with quantitative capacity estimations.

## Workflow

### Step 1: Identify what the user needs

Determine the scope from the user's request:

| Signal | Action |
|--------|--------|
| "HLD", "architecture", "system design", "high-level" | Produce an HLD — read `references/hld-template.md` |
| "LLD", "low-level", "detailed design", "class diagram" | Produce an LLD — read `references/lld-template.md` |
| "ADR", "decision record" | Produce an ADR using the template in `references/hld-template.md` (ADR section) |
| "estimate", "capacity", "back-of-envelope", "sizing" | Run estimations — read `references/estimation-cheatsheet.md` |
| "design doc", "design document", "tech spec" | Produce both HLD + LLD — read both templates |
| "Gate review", "proposal" | Produce HLD with executive summary and ADRs |

### Step 2: Gather requirements

Before writing anything, extract or ask for:

**Functional requirements (FR):**
- What does the system do? (use cases, user stories)
- Who are the actors? (users, external systems, admins)
- What are the core workflows?

**Non-functional requirements (NFR):**
- Performance: latency targets, throughput (RPS)
- Availability: SLO target (99.9%? 99.99%?)
- Scalability: expected users, growth rate
- Security: auth model, encryption, compliance (ANSSI, DISA STIG, etc.)
- Data: retention, backup, sovereignty constraints
- Deployment: cloud, on-prem, air-gapped, hybrid

If the user has already provided context (existing docs, previous conversations), use that instead of asking redundant questions.

### Step 3: Run back-of-envelope estimations

Read `references/estimation-cheatsheet.md` and compute:
- Traffic estimation (QPS, peak QPS)
- Storage estimation (per object x count x retention)
- Bandwidth estimation (QPS x avg response size)
- Cache sizing (hot data % x total data)
- Server count (peak QPS / QPS per server x 1.3 headroom)

Present estimations in a clear table. These numbers drive architecture decisions.

### Step 4: Write the document

Read the appropriate template from `references/` and produce the document.

**For HLD** -> read `references/hld-template.md`:
- Use C4 Level 1 (Context) and Level 2 (Container) diagrams
- Document architecture decisions as ADRs
- Include capacity estimations from Step 3
- Cover all NFRs explicitly
- Use Mermaid diagrams for visual representations

**For LLD** -> read `references/lld-template.md`:
- Use C4 Level 3 (Component) and Level 4 (Code) detail
- Include class/sequence diagrams (Mermaid)
- Define API contracts (REST/gRPC with request/response schemas)
- Specify database schemas with indexes
- Detail error handling and retry strategies

### Step 5: Output format

Default output is **Markdown** (`.md` file) with Mermaid diagrams inline.

If the user requests a Word document or formal deliverable, use the `docx` skill to convert the markdown to a professional `.docx` with proper formatting, headers, table of contents, and page numbers.

If the user requests a presentation (for Gate review), use the `pptx` skill.

### Step 6: Review checklist

Before delivering, verify against the relevant checklist:

**HLD checklist:**
- [ ] Context diagram (C4 L1) showing system boundaries
- [ ] Container diagram (C4 L2) showing major components
- [ ] NFRs explicitly addressed (performance, availability, security, scalability)
- [ ] Back-of-envelope estimations included
- [ ] Technology stack justified
- [ ] At least 1 ADR for each major architectural decision
- [ ] Data flow described
- [ ] Deployment architecture described
- [ ] Security model described
- [ ] Failure modes and mitigation described

**LLD checklist:**
- [ ] Component diagram (C4 L3) for each container
- [ ] API contracts defined (endpoints, methods, request/response)
- [ ] Database schema with indexes and relationships
- [ ] Sequence diagrams for critical flows
- [ ] Error handling strategy
- [ ] Class/module structure
- [ ] Caching strategy details
- [ ] Logging and observability hooks

## Key principles

- **Every decision needs a "why"** — no technology choice without justification
- **Tradeoffs are explicit** — use ATAM-style reasoning: "We chose X over Y because [quality attribute] is more important than [other attribute] in this context"
- **Numbers before architecture** — always estimate before designing. The numbers drive the design, not the other way around
- **Diagrams are mandatory** — use Mermaid for all diagrams (C4, sequence, ER, flowcharts)
- **Iterate, don't gold-plate** — a good design doc is a living document. Ship v1 fast, iterate based on feedback
