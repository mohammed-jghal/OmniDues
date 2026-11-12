# Architecture

OmniDues is a **modular monolith** with strict layer boundaries. The shape, in one line:

**Next.js web → FastAPI API → 106 service files → PostgreSQL 16**, with PgBouncer in front of the API only, standalone worker processes with command-line interfaces, and a transactional outbox feeding signed webhooks.

This document describes the sealed v5 shape. Wave 0a additions are noted where they change the shape. 5.0.1 is sealed (commit `e30aab9`, Entry 0263, 2026-11-12), with one documented exception — the sweep timing; nothing about 5.0.1 is presented as unsealed. Diagrams for this document live in [diagrams.md](diagrams.md) — diagrams 1, 2, 3, 6, 7, 8, 9, 14, and 15.

## Why a modular monolith — the decision and its alternative

The system is one deployable unit with strict internal layer boundaries, not a set of independently deployed services. The reason is stated plainly: the money invariants (derived balances, idempotent payments, atomic allocations) and the tenant-isolation invariants (one tenant context per transaction, enforced by the database) are easier to hold inside one transaction boundary than across service boundaries, where a partial failure would leave money in two states at once. The alternative — splitting into services first — was not chosen, because it would have traded the hardest correctness problem (money + tenancy in one transaction) for the hardest operations problem (distributed transactions, cross-service consistency) before any tenant existed. The cost of the choice is also stated: the monolith must be split later if one part needs to scale independently, and the load-test figures in [evidence-and-gates.md](evidence-and-gates.md) show the create path is already the bottleneck on demo hardware. That split is a Wave 1 / Wave 2 decision, not a v5 claim.

## Layers and their boundaries

- **Routers (49 modules).** Routers receive requests, validate input, and call services. They do not write SQL directly — a rule verified across the invoice, payment, collections, bank, dispute, and legal routers.
- **Services (106 files).** Services carry the domain logic: invoice balance and status derivation, idempotent payment creation with row locks, the collections state machine, bank-match evaluation, legal packs, and advisory credit-signal reads (kept as a unit separate from any signal mechanism).
- **Models (one file, 4,140 lines, 119 tables at seal).** The data map is deliberately kept in a single file so the whole schema can be read in one place. See [Data model](#data-model) below. 5.0.1 holds 120 tables (migration 0073 adds one), sealed in commit `e30aab9`.
- **Data access (async engine).** The tenant context is re-asserted at the start of every transaction. A missing tenant becomes a value that matches nothing — **fail-closed by design**.

Why the split is drawn here: routers are thin because a router bug must not be able to corrupt money — a router that validated input wrong could still only call a service, and the service owns the money rule and its tests (Money group, 95.9% at seal). Services are many (106 files) because each one owns one invariant: one service derives balances, another creates payments idempotently, another advances enrollments. A service that owns one invariant can be tested against that invariant alone, and a change to one invariant touches one file instead of a god-module that every change risks. The cost is indirection — a request crosses two layers before it touches data — and the benefit is that the money rules live in exactly one place each, where the gate can pin them.

What each layer is forbidden to do: a router never decides a money rule (that is a service decision, tested in the Money coverage group at 95.9% at seal); a service never decides a tenant boundary by itself (the database enforces it underneath, so a service bug cannot leak a row to another tenant — see [Tenancy](#tenancy)); no layer ever stores a balance as a writable field (balances are derived at read time from primitives, so a stale balance cannot exist as stored data).

## Request path — step by step, with failure modes

Every authenticated request passes the same gates:

1. **Authentication.** A Bearer JWT is decoded and checked; a missing, corrupt, or disabled token returns 401. A token version is compared against the user record, so any credential, role, or MFA change invalidates prior tokens. Failure mode: a stolen token stops working the moment the user changes a credential, a role, or MFA — the invalidation is by version bump, not by waiting for expiry.
2. **Tenant resolution.** The `X-Tenant-Id` header selects the tenant among the user's memberships: 403 for a tenant the user is not a member of, 400 for multiple memberships with no header. Platform-admin impersonation tokens are pinned to exactly one tenant. Failure mode: no header and several memberships is an explicit 400, not a guess at a default tenant — the system never silently picks a tenant for the caller.
3. **Per-endpoint authorization.** Each endpoint declares its required permission or role; a custom role **replaces** its system template entirely (it never merges with it), and unknown permissions are rejected at validation. Failure mode: a custom role that forgets a permission loses it — there is no silent inheritance from the template it replaced.
4. **Row-level security.** The session context is applied per transaction (see [Tenancy](#tenancy)). Failure mode: if the tenant context is missing, the query matches nothing — the request fails closed instead of returning another tenant's rows.
5. **Audit.** Impersonation actions are stamped with a platform-admin actor, so break-glass access is attributable after the fact.

The request path also passes through rate limiting (Backpressure), read-only ViewAs sessions, deprecation headers (RFC 8594; the deprecation log is empty), a NUL-byte guard (NUL bytes and non-terminating JSON numbers rejected), security headers, trace context, and CORS. Wave 0a added CSRF protection as an outermost middleware for browser sessions. The exact execution order of the middleware components is an implementation detail and is not asserted in this document.

## Boot discipline

The application **refuses to boot when the applied database migrations do not match the code's migration head**. There is an emergency escape for this check, and it is never set in a deployed environment. Missing core configuration (secret key, application database URL) also stops boot; SQLite is refused.

Boot refuses for a reason a reader should understand: the migration head is the schema the code was written against. Booting against an older schema would let the code write rows the schema cannot hold, or read columns that do not exist. Refusing to boot turns a silent data-corruption risk into a loud, immediate failure at startup — before any tenant data is touched. The same discipline applies to the encryption key: production boot fails without a configured key, so the system never starts in a state where it would store PII in plaintext by accident.

## Data model

119 tables in 14 groups at the v5 seal:

| Group | Tables | Group | Tables |
|---|---|---|---|
| Tenancy and identity | 23 | Integrations | 6 |
| Customers and tags | 3 | Compliance, privacy, audit, security | 12 |
| Invoices, credit notes, numbering, taxes, currencies | 9 | Regional e-invoicing | 4 |
| Payments, promises, mandates | 8 | AI and forecasting | 9 |
| Banks and PSP | 8 | Platform operations and documents | 6 |
| Collections | 7 | Credit | 8 |
| Disputes and legal | 7 | Statements, digests, communications | 9 |

What each group holds, in one line each: Tenancy and identity holds tenants, users, memberships, roles, and sessions. Customers and tags holds the debtor master data and its labels. Invoices, credit notes, numbering, taxes, and currencies holds the money primitives themselves. Payments, promises, and mandates holds recorded payments, payment promises, and debit mandates. Banks and PSP holds bank accounts, statements, and payment-provider records. Collections holds playbooks, enrollments, steps, and call outcomes. Disputes and legal holds disputes and the legal-escalation ladder. Integrations holds external-system bindings. Compliance, privacy, audit, and security holds the audit chain, consent, and retention records. Regional e-invoicing holds the ZATCA and DGI surfaces. AI and forecasting holds the forecasting inputs and outputs (advisory only). Platform operations and documents holds operational records and generated documents. Credit holds the advisory credit-signal records (never a limit, never enforced). Statements, digests, and communications holds the outbound-communication records — drafts, sends, and digests — with human approval recorded at each step.

### Tenancy

Isolation is built into the schema, not only enforced in code:

- **Composite foreign keys.** Child rows reference their parent with a composite `(tenant_id, X_id)` key, so a child row cannot point at another tenant's parent even if row-level security were bypassed.
- **Row-level security.** At the v5 seal: 118 tables under FORCE row-level security with 729 logical policies (953 physical rows including per-partition copies). The single table without FORCE RLS is the plans reference table: it carries no `tenant_id`, and the application role reads it only. In 5.0.1 (sealed), the census is 119 of 120 tables under FORCE RLS (733 logical policies, 957 physical rows); the plans table remains the single exception.
- **Three database roles, separated by function.** The application role cannot superuser, cannot bypass RLS, and owns nothing. The migrator role owns the schema. The backup role bypasses RLS but is select-only.
- **Per-transaction session context.** The tenant identity is set as a transaction-local configuration value at every transaction start.

The three layers of tenancy, and what each one catches: the permission check catches a user who should not see a screen at all; row-level security catches a query that would otherwise return another tenant's rows; the composite foreign key catches a write that would attach a child row to another tenant's parent even if both layers above were bypassed. A single layer failing does not leak data, because the next layer still holds. That is the whole design: defence in depth, with the database as the last line, not the first.

## Money rules — the invariants

- **Decimal only.** No floating-point number ever touches money; amounts are NUMERIC in the database and Decimal in code.
- **Balances are derived, never set by hand.** Balance = amount − payments − adjustments (`B = A − P − C`). Invoice financial status is derived from recorded primitives: a draft or cancelled invoice passes through as-is; a balance at or below zero is PAID; any payment or adjustment makes it PARTIALLY_PAID; otherwise OPEN.
- **Payments are idempotent and locked.** Creation carries a request fingerprint so a retried request cannot double-charge; invoice rows are locked for update during allocation; allocations and reversals are atomic.
- **Financial records are never hard-deleted.** Corrections happen through recorded adjustments and reversals.
- **Rounding (amended 2026-11-01, committed in the 5.0.1 seal).** ADR-012 was amended so foreign-exchange consolidation quantizes per currency with ROUND_HALF_UP before summation. The penalty engine's documented HALF_EVEN rounding for advisory, non-ledger figures is untouched; the divergence is recorded, not hidden.
- **No automated credit decisions, ever.** This is a binding architecture decision (ADR-002), not a missing feature. Credit signals are advisory reads only (see below).

Why derived balances matter: if a balance were a stored field, two writers could disagree about it — a payment recorded while a credit note is applied, for example — and the stored value would be wrong until someone noticed. A derived balance cannot go stale, because it is recomputed from the primitives every time it is read. The cost is read-time computation, which is part of why the create path is slow on demo hardware (see [evidence-and-gates.md](evidence-and-gates.md) — create p50 about 1,700 ms). The trade was made deliberately: correctness first, speed second, on the grounds that a wrong balance is worse than a slow one.

## Collections engine

- **Playbooks** define step sequences with conditions and quiet hours. Two seeded playbooks ship with the demo data: Standard (5 steps) and Ferme (7 steps).
- **Enrollments** are driven by a sweep worker that consumes events and advances steps under each tenant's timezone and quiet hours.
- **Promises** carry a legal window: the promised date must fall between today and today + 180 days, in the tenant's local date. Past dates are rejected.
- **Dispute interaction is fixed in code.** Opening a dispute pauses the active enrollment; resolving one resumes it only when no open dispute remains; the sweep skips enrollments with an open dispute. A paused-for-dispute enrollment is never silently resumed.

The sweep, in plain steps: it wakes on its schedule, takes the enrollments due for advancement, checks each one against the tenant's timezone and quiet hours (no step advances inside quiet hours), checks for an open dispute (skip if one exists), advances the playbook step, and records what it did. If a step is a reminder, it drafts the reminder — it does not send it. Sending is a human action, recorded separately. The full state machine is in [money-and-collections.md](money-and-collections.md) and diagram 4 in [diagrams.md](diagrams.md).

## Bank matching

1. Statements import with a per-entry fingerprint and parser logs, with the account binding checked.
2. Candidates are scored on amount, reference, and counterparty-name overlap.
3. Confirming, rejecting, or resolving a match is a human operator action gated by permission. **Money is never allocated automatically.**
4. Reconciliation closes the loop per account, including through a standalone reconciliation worker.

The fingerprint matters because the same statement file can be imported twice by mistake. A per-entry fingerprint makes the second import a duplicate the system recognises, instead of a second set of money movements. The scoring matters because a match is a suggestion, not a decision: amount, reference, and counterparty name each contribute evidence, and a human confirms or rejects the suggestion. No score, however high, allocates money on its own.

## Credit signals — advisory only

- Consent is an explicit opt-in action, never an assumption; withdrawal changes state; consent names its purpose, and any other purpose is refused.
- Aggregates are counts and k-anonymized sums only. Raw data never crosses tenants, and storage refuses rows below the anonymity threshold at write time.
- The network gate: until counsel clears it, the only state is NOT_CLEARED, which refuses before any extraction and logs the refusal in the append-only chain.
- Behavioural advisory per customer follows the same discipline: advisory configuration and advisory reads only. No automatic hold, limit, or block exists in code.

## Workers

Background work runs as standalone processes with `--once` / `--loop` command-line interfaces, each on its own database connection (migrations and copies connect directly to the database, never through the connection pooler):

| Worker | What it does | How it runs |
|---|---|---|
| Collections sweep | Advances playbook enrollments under tenant timezone and quiet hours; skips disputed enrollments | Standalone CLI; Compose service since wave 0a |
| Bank reconciliation | Closes the reconciliation loop per account | Standalone CLI |
| Digests | Produces periodic digest records | Standalone CLI |
| ERP sync | Synchronises with an ERP binding | Standalone CLI |
| Statements | Processes imported statements | Standalone CLI |
| Metering | Records per-tenant usage for plan enforcement | Standalone CLI |
| Webhook delivery | Delivers signed webhooks to receivers | Standalone CLI |
| Outbox publishing | Publishes the transactional outbox | Standalone CLI; also available as an optional in-application task |

- Retention is CLI-only (`--once`, with an explicit `--apply` required before anything is deleted or anonymized; the default is a dry-run evaluation).
- The outbox publisher is also available as an optional in-application task; in wave 0a the workers additionally run as Compose services.

Why workers are standalone processes with command-line interfaces instead of background threads inside the API: a worker that crashes must not take the API down with it, and a worker that is slow must not block requests. Each worker runs `--once` for a single pass an operator can watch, or `--loop` for continuous work, on its own database connection — so an operator can run one worker by hand to reproduce a failure, restart one worker without restarting the API, and see in the process list exactly which piece of background work is alive. The cost is operational surface (eight processes to supervise instead of one), which wave 0a pays with Compose services and an install script that fails loudly naming any worker not running.

Workers connect directly to the database, never through PgBouncer, for a specific reason: PgBouncer in transaction-pooling mode resets session state between transactions, and the workers rely on session state that must survive across their own work. The API, by contrast, sets its tenant context fresh at every transaction start, so it is safe behind the pooler. That split — API behind PgBouncer, workers direct — is deliberate, and it was proven in wave 0a by the RLS-through-PgBouncer test (6 / 6 passed).

## Observability

- **Structured logs** carry a trace identifier on every line; an incoming W3C `traceparent` header is accepted when it matches, otherwise an identifier is generated, and the effective identifier returns as a response header.
- **Exactly five metric series** are exposed: HTTP request totals, HTTP error totals, HTTP request duration, unpublished outbox count, and uptime seconds. No other series exist.
- **Health vs readiness are distinct.** The health endpoint is a static OK with no database touch. The readiness endpoint runs a database query **and** checks the migration head, returning 503 with a checks object otherwise.

Five metric series is a small number, and it is small on purpose: the system exposes what an operator needs to know whether it is up, whether it is erroring, whether it is slow, whether the outbox is draining, and how long it has been up — and nothing else. There is no per-tenant metric, no per-endpoint latency histogram, and no business metric exposed as a series. Those would be useful, and their absence is a stated gap in [operations-and-limits.md](operations-and-limits.md), not an oversight discovered later.

## Localization

Arabic, French, and English interfaces with strict right-to-left layout and Moroccan numeric formatting. All user-facing strings go through centralized translation keys; no screen is single-language by accident. The keys are centralized for one reason: a string that lives in exactly one place can be audited for all three languages at once, and a missing translation is visible as a missing key rather than as a screen that silently renders in the wrong language. Right-to-left is a layout property, not a per-screen choice, so a new screen inherits RTL the way it inherits the rest of the layout — no screen can accidentally ship left-to-right because its author forgot.

## Failure-mode catalogue — what breaks, and what happens then

| Failure | What the user sees | What the system does | What it does NOT do |
|---|---|---|---|
| Missing / corrupt / disabled token | 401 | Refuses before any domain work | Never serves a partial response |
| Tenant the user is not a member of | 403 | Refuses; no tenant data returned | Never falls back to another tenant |
| Several memberships, no tenant header | 400 | Refuses; asks for an explicit tenant | Never guesses a default tenant |
| Unknown permission on a custom role | Rejected at validation | Role cannot be saved with it | Never silently drops the permission |
| Missing tenant context in a transaction | Empty result | Query matches nothing (fail-closed) | Never returns another tenant's rows |
| Migration head mismatch at boot | Boot refusal | Process exits before serving | Never boots against the wrong schema |
| Missing secret key / database URL / encryption key at boot | Boot refusal | Process exits before serving | Never starts with a default secret |
| Duplicate payment request (same fingerprint) | Original result returned | No second payment created | Never double-charges |
| Two payments against one invoice at once | Second waits on the row lock | Serialised; second reads the updated balance | Never allocates against a stale balance |
| 30-digit amount | Rejected by the ledger guard | Loud rejection before the Decimal context | Never stores a wrong number (5.0.1 fix) |
| Statement file imported twice | Duplicate recognised | Second import flagged by fingerprint | Never creates a second set of movements |
| Dispute opened mid-collection | Enrollment pauses | Sweep skips the invoice | Never resumes silently |
| Outbox publisher stops | Unpublished-outbox metric rises | Events wait in the outbox (not lost) | Never delivers an event twice by retrying blindly — delivery is signed and recorded |
| Worker stops | Its work stops (see operations doc) | Nothing else compensates for it | Never silently reassigns its work to the API |

## Migrations — 71 at seal, 0073 in 5.0.1

Migrations are sequential and the application refuses to boot unless the applied set matches the code's head. At the v5 seal there were 71 migrations (last: `0071_dpa_suite.py`) — the figure 72 that appeared in earlier documentation was wrong, corrected here from `git ls-tree` at the seal commit `fbd8a99`. Migration 0072 (wave 0a) added ICE / tax-ID encryption; migration 0073 (sealed in 5.0.1, commit `e30aab9`) adds email verification — one new table, taking the census from 119 to 120 tables. The discipline is the point: a migration is a schema change with a number, a direction, and a test; there is no undocumented schema change, and the boot check makes an undocumented change impossible to deploy by accident.

## API versioning and deprecation

Deprecation headers follow RFC 8594, and the deprecation log is empty — no endpoint is currently deprecated. That is a fact about the current surface, not a promise: when an endpoint is deprecated, the header will say so and the log will record it. The API version string `3.0.1-rc1` is internal lineage, not a product claim, and is not presented as a release version anywhere in this repository.
