# Diagrams

All diagrams are Mermaid (GitHub renders Mermaid natively). Every label is in English. Every element matches a verified fact in the other documents of this repository; where a flow detail is not in the record, it is left out rather than invented. **v3 adds diagrams 6–12** to the five diagrams of v2: the money lifecycle, payment recording, bank matching, workers and outbox, authentication and session lifecycle, audit chain and restore, and the gate / seal lifecycle.

What is intentionally **not** shown here:

- Live PSP, bank-feed, WhatsApp Cloud, and email-delivery flows are Tier 3 gated and not live, so they appear only as gated external systems in the context diagram, never as working flows.
- The exact execution order of the middleware components is not asserted (see [architecture.md](architecture.md)).
- No production deployment topology is shown, because no production deployment exists; the only running copy is a trial on the owner's PC (loopback only).

## 1. System context

```mermaid
flowchart TB
    Collector["Tenant users<br/>Collector · Finance · Admin · Viewer<br/>RBAC roles, custom roles replace templates"]
    PlatformAdmin["Platform admin<br/>Break-glass, pinned to one tenant"]
    OmniDues["OmniDues<br/>Multi-tenant B2B receivables platform<br/>Customers · Invoices · Payments<br/>Collections · Disputes · Legal escalation"]
    Bank["Banks<br/>Statements and feeds"]
    PSP["Payment service providers<br/>Payment links"]
    WhatsApp["WhatsApp<br/>Manual drafts today (operator sends)<br/>Cloud API: Tier 3, not live"]
    Email["Email provider<br/>Verification and notices<br/>Not live — new self-registration<br/>cannot complete verification yet"]
    ZATCA["ZATCA e-invoicing<br/>Specifications and tooling only<br/>Production credentials: Tier 3"]
    DGI["DGI (Morocco) e-invoicing<br/>Design surface only<br/>Channel frozen off at schema level"]
    Trial["Trial deployment<br/>Owner's PC only, loopback<br/>Demo tenant, fictional data"]

    Collector --> OmniDues
    PlatformAdmin --> OmniDues
    OmniDues -.->|"Tier 3 gate — not live"| Bank
    OmniDues -.->|"Tier 3 gate — not live"| PSP
    OmniDues -.->|"Manual drafts only"| WhatsApp
    OmniDues -.->|"Tier 3 gate — not live"| Email
    OmniDues -.->|"Tier 3 gate — not live"| ZATCA
    OmniDues -.->|"Frozen off"| DGI
    Trial -.->|"Hosts"| OmniDues
```

## 2. Containers

```mermaid
flowchart TB
    Browser["Web browser<br/>Next.js frontend<br/>Arabic · French · English, strict RTL"]
    API["FastAPI API<br/>49 routers → 106 service files<br/>Decimal-only money · idempotent payments"]
    Workers["Worker processes (8)<br/>Collections sweep · Bank reconciliation<br/>Digests · ERP sync · Statements<br/>Metering · Webhook delivery · Outbox publishing<br/>Standalone CLIs; Compose services since wave 0a"]
    Outbox["Transactional outbox<br/>Signed webhooks"]
    PgBouncer["PgBouncer<br/>In front of the API only<br/>Workers connect directly to the database"]
    DB[("PostgreSQL 16<br/>119 tables at the v5 seal · 71 migrations<br/>FORCE RLS · AES-256-GCM PII<br/>Hash-chained audit tables")]
    Receivers["External webhook receivers<br/>Signed delivery"]

    Browser --> API
    API --> PgBouncer
    PgBouncer --> DB
    Workers --> DB
    API --> Outbox
    Outbox --> Receivers
```

## 3. Tenant isolation — request flow

```mermaid
flowchart TB
    Req["Incoming request<br/>Bearer JWT + X-Tenant-Id header"]
    Auth["1 · Authenticate<br/>Decode JWT · check token version<br/>Any credential, role, or MFA change<br/>invalidates prior tokens"]
    Tenant["2 · Resolve tenant<br/>X-Tenant-Id among the user's memberships<br/>403 if not a member<br/>400 if several memberships and no header"]
    Perm["3 · Authorize endpoint<br/>Declared permission or role<br/>Custom roles replace, never merge,<br/>their system template"]
    Ctx["4 · Set transaction context<br/>Tenant identity set transaction-local<br/>at every transaction start"]
    RLS["5 · Enforce row-level security<br/>118 of 119 tables under FORCE RLS at the v5 seal<br/>729 logical policies at seal<br/>119 of 120 tables in sealed 5.0.1"]
    FK["6 · Schema backstop<br/>Composite foreign keys (tenant_id, X_id)<br/>A child row cannot reference<br/>another tenant's parent"]
    Audit["7 · Audit<br/>Impersonation stamped with<br/>platform-admin actor"]
    Fail["Fail-closed<br/>Missing tenant matches nothing"]

    Req --> Auth --> Tenant --> Perm --> Ctx --> RLS --> FK --> Audit
    Ctx -.-> Fail
```

## 4. Collections workflow

```mermaid
flowchart TB
    Invoice["Invoice issued<br/>Balance derived: B = A − P − C<br/>Status derived: OPEN · PARTIALLY_PAID · PAID"]
    Overdue["Invoice becomes overdue<br/>Sweep enrolls it in a playbook"]
    Playbook["Playbook steps<br/>Standard (5 steps) · Ferme (7 steps)<br/>Tenant timezone · quiet hours"]
    Draft["Reminder drafted<br/>Draft-with-human-approval<br/>Operator reviews and sends<br/>No automated sending"]
    Contact["Contact logged<br/>Call outcome recorded<br/>Manual WhatsApp draft link"]
    Promise["Promise recorded<br/>Promised date: today to today + 180 days<br/>Tenant local date · past dates rejected"]
    Payment["Payment recorded<br/>Idempotent · row-locked · atomic allocation<br/>Cheque / commercial paper recordable<br/>(planned, wave 0b)"]
    Paid["Balance at or below zero → PAID<br/>Financial records are never hard-deleted"]
    Dispute["Dispute opens<br/>Enrollment pauses automatically"]
    Resume["Dispute resolved<br/>Resume only when no dispute remains open"]

    Invoice --> Overdue --> Playbook --> Draft --> Contact --> Promise --> Payment --> Paid
    Contact -.->|"Dispute opened"| Dispute
    Dispute -.->|"Resolved, none open"| Resume
    Resume -.-> Playbook
```

## 5. Data flow and encryption

```mermaid
flowchart TB
    subgraph Input["Tenant data in"]
        PII["PII: email · phone · ICE · tax IDs"]
        Money["Money: amounts as Decimal / NUMERIC<br/>No floating point touches money"]
        Docs["Documents and statements<br/>Per-entry fingerprint"]
    end
    subgraph Protect["Protection at rest"]
        Enc["AES-256-GCM ciphertext<br/>Key-provider abstraction<br/>Environment / KMS / vault by configuration<br/>Boot fails without a configured key"]
        Blind["Blind indexes for search<br/>Email · phone · customers.ice<br/>No blind index where search is not required"]
        Mask["Field masking<br/>Viewer and masked custom roles"]
    end
    subgraph Record["Record and proof"]
        AuditChain["Audit chain<br/>Append-only · hash-chained<br/>Tampering breaks the chain"]
        Backup["Backup (select-only role)<br/>Restore drill: about 73 s<br/>against a 15-minute target<br/>Backup encryption: NOT shipped<br/>(owner decision)"]
    end
    KMS["Key provider<br/>Environment / KMS / vault by configuration<br/>Live KMS: Tier 3, not live"]

    PII --> Enc
    Enc --> Blind
    PII --> Mask
    Money --> AuditChain
    Docs --> AuditChain
    AuditChain --> Backup
    Enc -.->|"Key provider"| KMS
```

## 6. Money lifecycle — invoice state machine

```mermaid
stateDiagram-v2
    [*] --> DRAFT: Invoice created
    DRAFT --> OPEN: Issued
    DRAFT --> CANCELLED: Cancelled
    OPEN --> PARTIALLY_PAID: Payment or adjustment recorded
    PARTIALLY_PAID --> PARTIALLY_PAID: Further payment or adjustment
    PARTIALLY_PAID --> PAID: Balance at or below zero
    OPEN --> PAID: Balance at or below zero
    OPEN --> OVERDUE: Past due date, balance above zero
    PARTIALLY_PAID --> OVERDUE: Past due date, balance above zero
    OVERDUE --> PARTIALLY_PAID: Payment recorded, balance above zero
    OVERDUE --> PAID: Balance at or below zero
    PAID --> [*]: Closed — never hard-deleted
    CANCELLED --> [*]: Closed — never hard-deleted
    note right of OPEN
        Status is derived, never set by hand:
        B = A − P − C
        B ≤ 0 → PAID
        Any payment or adjustment → PARTIALLY_PAID
        Otherwise → OPEN
    end note
```

## 7. Payment recording — sequence

```mermaid
sequenceDiagram
    actor Collector
    participant UI as Web UI
    participant API as FastAPI API
    participant DB as PostgreSQL 16
    Collector->>UI: Record payment (amount, date, reference)
    UI->>API: POST payment + request fingerprint
    API->>API: Check fingerprint — duplicate request?
    alt Duplicate fingerprint
        API-->>UI: Return original result — no double charge
    else New request
        API->>DB: Begin transaction, set tenant context
        API->>DB: Lock invoice row FOR UPDATE
        API->>DB: Validate amount (Decimal, ledger guard)
        API->>DB: Insert payment + atomic allocation
        API->>DB: Write audit entry (hash-chained)
        API->>DB: Commit — balance derived on next read
        DB-->>API: Committed
        API-->>UI: Payment recorded, new balance
    end
    note over API,DB: A PAID call outcome is rejected<br/>until a payment is recorded first.
```

## 8. Bank matching — from statement to reconciliation

```mermaid
flowchart TB
    Import["Statement import<br/>Per-entry fingerprint<br/>Parser logs · account binding checked"]
    Dup{"Duplicate<br/>fingerprint?"}
    Score["Candidate scoring<br/>Amount · reference<br/>Counterparty-name overlap"]
    Suggest["Match suggested<br/>A suggestion, never a decision"]
    Human["Human operator<br/>Confirm · reject · resolve<br/>Permission-gated"]
    Allocate["Money allocated<br/>Only after human confirmation"]
    Recon["Reconciliation<br/>Closes the loop per account<br/>Standalone worker"]
    Reject["Rejected match<br/>Recorded, not deleted"]

    Import --> Dup
    Dup -->|"Yes — duplicate"| Reject
    Dup -->|"No"| Score --> Suggest --> Human
    Human -->|"Confirm"| Allocate --> Recon
    Human -->|"Reject / resolve"| Reject
```

## 9. Workers and outbox

```mermaid
flowchart TB
    subgraph App["FastAPI API"]
        Txn["Business transaction<br/>Writes domain rows + outbox row<br/>in the same transaction"]
    end
    subgraph Workers["Worker processes (8)"]
        Sweep["Collections sweep"]
        Recon["Bank reconciliation"]
        Digest["Digests"]
        ERP["ERP sync"]
        Stmt["Statements"]
        Meter["Metering"]
        Hook["Webhook delivery"]
        Out["Outbox publishing"]
    end
    DB[("PostgreSQL 16<br/>Workers connect directly<br/>Never through PgBouncer")]
    Recv["External webhook receivers<br/>Signed delivery"]

    Txn --> DB
    Workers --> DB
    Out --> Recv
    Hook --> Recv
    Txn -.->|"Outbox row"| Out
    note right of Out
        Outbox publishing is also available
        as an optional in-application task.
        Retention is CLI-only: --once,
        --apply required before any
        deletion or anonymization.
    end note
```

## 10. Authentication and session lifecycle

```mermaid
flowchart TB
    Reg["Register<br/>No session created<br/>No 409 oracle"]
    Verify["Verify email<br/>5.0.1, sealed<br/>Not completable until<br/>live email or operator stamp"]
    Login["Login<br/>Bearer JWT issued<br/>Token version = current"]
    MFA["MFA (optional)<br/>Backup codes<br/>5 failures / 15 min throttle<br/>5.0.1, sealed"]
    Work["Work under tenant context<br/>JWT + X-Tenant-Id<br/>Version checked per request"]
    Change["Credential / role / MFA change<br/>Token version bumped"]
    Dead["Prior tokens dead<br/>Next request fails version check"]
    Logout["Logout<br/>Global version bump<br/>All tokens dead everywhere"]
    Imp["Platform-admin impersonation<br/>Pinned to one tenant<br/>Security changes forbidden<br/>Stamped in audit chain"]

    Reg --> Verify --> Login --> MFA --> Work
    Work --> Change --> Dead
    Work --> Logout
    Work -.-> Imp
```

## 11. Audit chain and restore

```mermaid
flowchart TB
    subgraph Chain["Audit chain — append-only, hash-chained"]
        G["Genesis entry<br/>Defined genesis hash"]
        E1["Entry N<br/>Hash = H(previous hash + framed fields)"]
        E2["Entry N+1<br/>Hash = H(previous hash + framed fields)"]
        Verify["Verification<br/>Walk from genesis, recompute every hash<br/>First mismatch names the broken entry<br/>Every later entry untrusted"]
    end
    subgraph Backup["Backup and restore"]
        Role["Backup role<br/>Select-only · bypasses RLS for reads<br/>Can write nothing"]
        Dump["Logical backup<br/>Demo drill: 1.8 s backup"]
        Restore["Restore into fresh database<br/>Demo drill: 71.4 s restore<br/>About 73 s total vs 15-min target<br/>Row counts matching"]
        Enc["Backup encryption<br/>NOT shipped — owner decision"]
    end
    G --> E1 --> E2 --> Verify
    E2 --> Role
    Role --> Dump --> Restore
    Restore -.-> Enc
```

## 12. Gate and seal lifecycle

```mermaid
flowchart TB
    V5["v5 — SEALED<br/>Gate 5,192 passed / 0 failed<br/>Coverage 84.3% / 96.8%<br/>Entry 0262"]
    W0a["Wave 0a — COMMITTED<br/>Commit 5d029b5, 98 files<br/>Gate 3: 1 timing failure<br/>384 s vs 300 s, counts exact<br/>Documented exception"]
    V501["5.0.1 — SEALED 2026-11-12<br/>Commit e30aab9 · Entry 0263<br/>P0 ×3 · security P1 ×6<br/>Correctness P1 · UX merge<br/>Migration 0073"]
    G501b["501b — RED<br/>3 failed / 5,298 passed<br/>Sweep 566.89 s"]
    G501c["501c — RED<br/>2 failed / 5,300 passed<br/>Sweep 378.04 s"]
    G501d["501d — RED, last gate<br/>1 failed / 5,301 passed<br/>Sweep 369.65 s vs 300 s<br/>Counts exact · solo 101.95 s<br/>Timing only — the seal exception"]
    D03["D03 — DECIDED 2026-11-12<br/>Owner: seal with documented exception<br/>Commit e30aab9 · Entry 0263"]
    Sealed["Sealed: single commit + seal entry<br/>5.0.1 sealed with the exception stated<br/>Any functional failure reopens the seal"]
    Held["Hold was the alternative — NOT taken<br/>The decision was extend + seal"]

    V5 --> W0a --> V501 --> G501b --> G501c --> G501d --> D03
    D03 --> Sealed
    D03 -.-> Held
```

## 13. RBAC and permissions

```mermaid
flowchart TB
    User["User<br/>Member of one or more tenants"]
    Role["Role<br/>System role OR custom role<br/>Custom roles REPLACE their template<br/>Never merge"]
    Owner["OWNER<br/>Everything in the tenant"]
    Admin["ADMIN<br/>Tenant administration"]
    Collector["COLLECTOR<br/>Queue · call outcomes<br/>Promises · drafts"]
    Finance["FINANCE<br/>Payments · adjustments<br/>Bank matching · reconciliation"]
    Viewer["VIEWER<br/>Read-only · masked fields"]
    Perm["Permission check<br/>Per endpoint, before domain work<br/>46-entry catalog at seal<br/>Unknown permission rejected"]
    Mask["Field masking<br/>Viewer + masked custom roles<br/>Row visible, field hidden"]

    User --> Role
    Role --> Owner
    Role --> Admin
    Role --> Collector
    Role --> Finance
    Role --> Viewer
    Role --> Perm
    Perm --> Mask
```

## 14. API surface

```mermaid
flowchart TB
    Clients["Clients<br/>Browser (cookie + CSRF)<br/>Programmatic (Bearer JWT)"]
    Surface["API surface at the v5 seal<br/>296 paths / 363 operations<br/>Committed OpenAPI snapshot<br/>Live: 296 / 365 — the +2 is<br/>two HEAD health operations<br/>5.0.1 seal: 300 paths / 367 operations"]
    Areas["Areas covered<br/>Customers · Invoices · Payments<br/>Collections · Disputes · Legal<br/>Banks · Matching · Reconciliation<br/>Credit reads (advisory only)<br/>Integrations · Operations · Documents"]
    Gate["Per-endpoint authorization<br/>Declared permission or role<br/>Checked before domain work"]
    Rate["Rate limits<br/>Per IP + per token per minute<br/>Per registration email per hour<br/>MFA throttle 5 / 15 min (5.0.1, sealed)"]
    Dep["Deprecation<br/>RFC 8594 headers<br/>Deprecation log: EMPTY<br/>No endpoint deprecated"]

    Clients --> Surface --> Areas
    Surface --> Gate --> Rate
    Surface --> Dep
```

## 15. Data-model groups

```mermaid
flowchart TB
    subgraph Core["Money core"]
        Inv["Invoices, credit notes<br/>Numbering, taxes, currencies<br/>9 tables"]
        Pay["Payments, promises, mandates<br/>8 tables"]
        Bank["Banks and PSP<br/>8 tables"]
    end
    subgraph Work["Collections work"]
        Coll["Collections<br/>Playbooks · enrollments · steps<br/>Call outcomes — 7 tables"]
        Disp["Disputes and legal<br/>OPEN / RESOLVED / CLOSED only<br/>7 tables"]
    end
    subgraph Trust["Identity and evidence"]
        Ten["Tenancy and identity<br/>23 tables"]
        Comp["Compliance, privacy, audit, security<br/>12 tables — hash-chained audit"]
        Cred["Credit — advisory only<br/>8 tables — never a limit"]
    end
    subgraph Rest["Supporting groups"]
        Cust["Customers and tags — 3"]
        Int["Integrations — 6"]
        Einv["Regional e-invoicing — 4<br/>ZATCA specs · DGI frozen off"]
        AI["AI and forecasting — 9<br/>Advisory only"]
        Ops["Platform operations, documents — 6"]
        Comm["Statements, digests, communications — 9"]
    end
    Ten --> Cust
    Cust --> Inv
    Inv --> Pay
    Pay --> Bank
    Inv --> Coll
    Coll --> Disp
    Disp -.->|"Pauses enrollment"| Coll
    Ten --> Comp
    Comp -.->|"Consent gates"| Cred
```

## 16. Trial deployment

```mermaid
flowchart TB
    PC["Owner's PC<br/>Loopback only — PC browser only<br/>Not reachable from a phone"]
    subgraph Compose["Compose — 13 services"]
        Web["Web (host process)<br/>Excluded from Compose<br/>Boot in Compose UNPROVEN"]
        API["FastAPI API<br/>Behind PgBouncer"]
        PgB["PgBouncer<br/>API only"]
        Work["8 workers<br/>Direct database connections"]
        DB[("PostgreSQL 16")]
    end
    Demo["Demo tenant<br/>Demo Distribution SARL<br/>Fictional data — 14 invoices"]
    Nope["NOT present<br/>No load balancer · no replica<br/>No staging · no production<br/>No live CI · no status page"]

    PC --> Compose
    Compose --> Demo
    API --> PgB
    PgB --> DB
    Work --> DB
    Web -.->|"Host process, not Compose"| API
    Compose -.-> Nope
```

## 17. Promise and dispute states

```mermaid
stateDiagram-v2
    [*] --> Enrolled: Invoice overdue, sweep enrolls
    Enrolled --> StepAdvanced: Sweep advances step (outside quiet hours)
    StepAdvanced --> Enrolled: Next step due
    Enrolled --> PromiseMade: Call outcome PROMISE (date T to T+180)
    PromiseMade --> Paid: Payment recorded, balance ≤ 0
    PromiseMade --> Enrolled: Promise date passes unpaid — back to queue
    Enrolled --> Disputed: Dispute opens — enrollment PAUSES
    Disputed --> Enrolled: Dispute resolved, none open — resume at same step
    Enrolled --> Paid: Balance at or below zero
    Paid --> [*]: Closed — never hard-deleted
    note right of Disputed
        The pause freezes state.
        It never resets the step.
        The sweep skips disputed
        enrollments entirely.
    end note
```

## 18. FX rounding — HALF_UP before summation

```mermaid
flowchart LR
    A["Amount in currency X<br/>Unrounded"]
    B["Quantize per currency<br/>ROUND_HALF_UP<br/>1,000.005 → 1,000.01"]
    C["Amount in currency Y<br/>Unrounded"]
    D["Quantize per currency<br/>ROUND_HALF_UP<br/>2,000.004 → 2,000.00"]
    E["Sum the ROUNDED figures<br/>1,000.01 + 2,000.00 = 3,000.01"]
    F["Ledger figure<br/>ADR-012 amended 2026-11-01<br/>Committed in the 5.0.1 seal (e30aab9)"]
    G["Penalty engine — SEPARATE<br/>HALF_EVEN, advisory only<br/>Never enters the ledger"]

    A --> B --> E --> F
    C --> D --> E
    G -.->|"Different rule, different figures"| F
```

## 19. Gate timeline

```mermaid
timeline
    title Gates — v5 seal to 501d
    2026-10-18 : v4 sealed (2,602 / 0)
    2026-10-22 : Rename sealed — OmniDues (surface only)
    v5 seal : 5,192 passed / 0 failed in 3,398.02 s — SEALED
    Wave 0a gate 3 : 1 timing failure — 384 s vs 300 s, counts exact — COMMITTED with exception
    5.0.1 first gate : 89 failed, 4,800 passed, 412 errors — RED
    501b : 3 failed / 5,298 passed — sweep 566.89 s — RED
    501c : 2 failed / 5,300 passed — sweep 378.04 s — RED
    501d : 1 failed / 5,301 passed — sweep 369.65 s — RED, timing only
    2026-11-12 : 5.0.1 SEALED (e30aab9, Entry 0263) — D03 decided, exception stated
```

## 20. Commercial readiness — desk assessment 513 / 1000

```mermaid
flowchart TB
    subgraph Eng["Engineering cluster — 666.7 (4,000 / 6)"]
        T["Test quality — 720"]
        D["Documentation — 700"]
        I["Tenant isolation — 700"]
        S["Security — 680"]
        A["Architecture — 640"]
        F["Functional — 560"]
    end
    subgraph Prod["Product / commercial cluster — 381.4 (2,670 / 7, incl. UX)"]
        U["UX — 600"]
        R["Reliability / DR — 520"]
        P["Performance — 470"]
        C["Compliance — 380"]
        O["Operations — 350"]
        G["Integrations — 200"]
        M["Commercial — 150"]
    end
    Verdict["Overall 513 / 1000 (unweighted mean)<br/>FAIL for production release<br/>Conditional PASS for controlled pilot only<br/>Desk assessment — NOT a live audit"]
    Eng --> Verdict
    Prod --> Verdict
```
