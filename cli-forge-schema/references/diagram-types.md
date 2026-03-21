# Mermaid Diagram Types — GitHub-Tested Reference

Every example below has been validated for GitHub rendering. Copy-paste safe.

---

## 1. Flowchart

Best for: processes, algorithms, decision trees.

```mermaid
flowchart TB
    A([Start]) --> B{Decision?}
    B -->|Yes| C[Action 1]
    B -->|No| D[Action 2]
    C --> E([End])
    D --> E
```

**Node shapes:**
| Shape | Syntax | Use for |
|-------|--------|---------|
| Rectangle | `[text]` | Process / action |
| Rounded | `(text)` | Generic step |
| Stadium | `([text])` | Start / end |
| Diamond | `{text}` | Decision |
| Cylinder | `[(text)]` | Database |
| Subroutine | `[[text]]` | External call |
| Parallelogram | `[/text/]` | Input / output |
| Hexagon | `{{text}}` | Preparation |
| Trapezoid | `[/text\]` | Manual operation |

**Edge styles:**
| Syntax | Meaning |
|--------|---------|
| `-->` | Solid arrow |
| `-.->` | Dotted arrow |
| `==>` | Thick arrow |
| `--text-->` | Labeled arrow |
| `~~~` | Invisible link (layout only) |

**Subgraphs:**
```mermaid
flowchart TB
    subgraph backend["Backend"]
        api[API Gateway]
        svc[Service]
        db[(Database)]
        api --> svc --> db
    end
    subgraph frontend["Frontend"]
        app[Web App]
    end
    app -->|REST| api
```

---

## 2. Sequence Diagram

Best for: interactions between actors/systems, API flows, message passing.

```mermaid
sequenceDiagram
    actor User
    participant API
    participant DB

    User->>API: POST /orders
    activate API
    API->>DB: INSERT order
    DB-->>API: OK
    API-->>User: 201 Created
    deactivate API
```

**Message types:**
| Syntax | Style |
|--------|-------|
| `->>` | Solid with arrowhead |
| `-->>` | Dashed with arrowhead |
| `--)` | Solid with open arrowhead (async) |
| `--)` | Dashed with open arrowhead (async) |
| `-x` | Solid with cross (lost message) |

**Features that work on GitHub:**
- `activate` / `deactivate` — lifeline activation
- `Note over A,B: text` — notes
- `loop`, `alt`, `opt`, `par`, `critical` — combined fragments
- `rect rgb(240,240,240)` — background highlight
- `autonumber` — message numbering

---

## 3. Class Diagram

Best for: object structures, interfaces, relationships.

```mermaid
classDiagram
    class Order {
        +String id
        +OrderStatus status
        +List~Item~ items
        +total() Money
        +cancel() void
    }

    class OrderStatus {
        <<enumeration>>
        PENDING
        CONFIRMED
        CANCELLED
    }

    class Repository {
        <<interface>>
        +save(Order) void
        +findById(String) Order
    }

    Order --> OrderStatus
    Repository ..> Order : manages
```

**Relationships:**
| Syntax | Meaning |
|--------|---------|
| `-->` | Association |
| `..>` | Dependency |
| `--|>` | Inheritance |
| `..|>` | Implementation |
| `--*` | Composition |
| `--o` | Aggregation |

---

## 4. State Diagram

Best for: lifecycles, state machines, workflows with conditions.

```mermaid
stateDiagram-v2
    [*] --> Draft
    Draft --> Review : submit
    Review --> Approved : approve
    Review --> Draft : request changes
    Approved --> Published : publish
    Published --> [*]

    state Review {
        [*] --> Pending
        Pending --> InReview : assign reviewer
        InReview --> Pending : reviewer unavailable
    }
```

**Tips:**
- Use `stateDiagram-v2` (not v1)
- Composite states for complex sub-flows
- `[*]` for start and end pseudo-states

---

## 5. ER Diagram

Best for: data models, database schemas, entity relationships.

```mermaid
erDiagram
    USER ||--o{ ORDER : places
    ORDER ||--|{ LINE_ITEM : contains
    PRODUCT ||--o{ LINE_ITEM : "is in"
    USER {
        uuid id PK
        string name
        string email UK
    }
    ORDER {
        uuid id PK
        uuid user_id FK
        date created_at
    }
```

**Cardinality:**
| Syntax | Meaning |
|--------|---------|
| `\|\|--\|\|` | One to one |
| `\|\|--o{` | One to many |
| `o{--o{` | Many to many |
| `\|\|--o\|` | One to zero or one |

---

## 6. Gantt Chart

Best for: project timelines, milestones, task dependencies.

```mermaid
gantt
    title Project Timeline
    dateFormat YYYY-MM-DD
    axisFormat %b %d

    section Design
    Research       :done, des1, 2025-01-01, 7d
    Prototype      :active, des2, after des1, 5d

    section Build
    Backend        :crit, dev1, after des2, 14d
    Frontend       :dev2, after des2, 10d

    section Launch
    Testing        :test, after dev1 dev2, 5d
    Deploy         :milestone, after test, 0d
```

**Status keywords:** `done`, `active`, `crit` (critical path)

---

## 7. Pie Chart

Best for: proportions, distributions. Max 7 slices.

```mermaid
pie title Traffic by Source
    "Organic" : 45
    "Direct" : 25
    "Social" : 15
    "Referral" : 10
    "Other" : 5
```

---

## 8. Mind Map

Best for: brainstorming, hierarchies, concept organization.

```mermaid
mindmap
    root((Project))
        Backend
            API
            Database
            Auth
        Frontend
            Web App
            Mobile
        Infrastructure
            CI/CD
            Monitoring
```

**Tips:**
- Max 4 levels deep
- Keep branch labels short (2-3 words)
- Use `(( ))` for root, plain text for branches

---

## 9. Timeline

Best for: chronological events, release history, project phases.

```mermaid
timeline
    title Release History
    2024 Q1 : v1.0 MVP
             : Initial launch
    2024 Q2 : v1.1 Performance
             : Cache layer added
    2024 Q3 : v2.0 Rewrite
             : New architecture
    2024 Q4 : v2.1 Scale
             : Multi-region support
```

---

## 10. Quadrant Chart

Best for: priority matrices, comparison, positioning.

```mermaid
quadrantChart
    title Effort vs Impact
    x-axis Low Effort --> High Effort
    y-axis Low Impact --> High Impact
    quadrant-1 Plan carefully
    quadrant-2 Do first
    quadrant-3 Eliminate
    quadrant-4 Delegate
    Feature A: [0.2, 0.8]
    Feature B: [0.7, 0.9]
    Feature C: [0.3, 0.2]
    Feature D: [0.8, 0.3]
```

---

## 11. Git Graph

Best for: branching strategies, merge workflows.

```mermaid
gitGraph
    commit id: "init"
    branch feature
    checkout feature
    commit id: "feat-1"
    commit id: "feat-2"
    checkout main
    merge feature id: "merge-feat"
    commit id: "release"
```

---

## 12. Sankey Diagram

Best for: flow volumes, budget allocation, traffic distribution.

```mermaid
sankey-beta
    Source A,Target X,30
    Source A,Target Y,20
    Source B,Target X,15
    Source B,Target Z,25
    Target X,Final,45
    Target Y,Final,20
    Target Z,Final,25
```

---

## Color Palette (GitHub-safe)

### Functional palette (5 colors max per diagram)

```
classDef primary fill:#4A90D9,stroke:#2C6CB0,color:#fff
classDef success fill:#27AE60,stroke:#1E8449,color:#fff
classDef warning fill:#F39C12,stroke:#D68910,color:#fff
classDef danger fill:#E74C3C,stroke:#C0392B,color:#fff
classDef neutral fill:#ECF0F1,stroke:#BDC3C7,color:#2C3E50
```

### Pastel palette (softer, for dense diagrams)

```
classDef blue fill:#D6EAF8,stroke:#85C1E9,color:#1B4F72
classDef green fill:#D5F5E3,stroke:#82E0AA,color:#186A3B
classDef yellow fill:#FEF9E7,stroke:#F9E79F,color:#7D6608
classDef red fill:#FADBD8,stroke:#F1948A,color:#922B21
classDef grey fill:#F2F3F4,stroke:#D5D8DC,color:#2C3E50
```

### Dark-mode friendly palette

```
classDef dm_blue fill:#2980B9,stroke:#1A5276,color:#fff
classDef dm_green fill:#1ABC9C,stroke:#117A65,color:#fff
classDef dm_orange fill:#E67E22,stroke:#A04000,color:#fff
classDef dm_red fill:#C0392B,stroke:#78281F,color:#fff
classDef dm_grey fill:#34495E,stroke:#1C2833,color:#ECF0F1
```

---

## GitHub Rendering Limits

| Constraint | Limit |
|-----------|-------|
| Max edges | 280 |
| Max characters per block | ~2000 for reliable rendering |
| Max practical nodes | 50 per diagram |
| Nesting depth (subgraphs) | 3 levels max |
| FontAwesome icons | Not supported |
| Click events | Not supported |
| Tooltips | Not supported |
| Custom fonts | Ignored |
| Themes via init | Only `default`, `dark`, `forest`, `neutral`, `base` |
