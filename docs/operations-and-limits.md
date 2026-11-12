# Operations and limits — deep dive

This document covers how the system runs, what an operator can observe, how backup and restore work, and — with equal weight — what is deliberately not present. It is the companion to [architecture.md](architecture.md) and [security.md](security.md). Diagrams 9 and 11 in [diagrams.md](diagrams.md) show the workers and the audit/backup flows.

## Boot discipline

The application refuses to boot when the applied database migrations do not match the code's migration head. There is an emergency escape for this check, and it is never set in a deployed environment. Missing core configuration (secret key, application database URL) also stops boot; SQLite is refused. Production boot fails without a configured encryption key.

The order of checks matters: the system fails at startup, before any tenant data is touched, rather than failing mid-request with a half-written row. An operator who sees a boot refusal knows exactly what to do — apply the missing migration, or supply the missing configuration — instead of debugging a data-corruption incident that a successful boot would have created.

## Workers — the eight processes

Background work runs as standalone processes with `--once` / `--loop` command-line interfaces, each on its own database connection, connecting directly to the database and never through PgBouncer:

| Worker | What it does | Failure mode if it stops |
|---|---|---|
| Collections sweep | Advances playbook enrollments under tenant timezone and quiet hours; skips disputed enrollments | Enrollments stop advancing; the queue goes stale; no reminder is drafted |
| Bank reconciliation | Closes the reconciliation loop per account | Statements pile up unmatched; reconciliation lags |
| Digests | Produces periodic digest records | Operators lose their periodic summary; no money impact |
| ERP sync | Synchronises with an ERP binding | ERP data drifts from the system |
| Statements | Processes imported statements | Imports queue; matching waits |
| Metering | Records per-tenant usage for plan enforcement | Plan enforcement (the 5,000/month GROWTH cap, for example) stops being measured |
| Webhook delivery | Delivers signed webhooks to receivers | External receivers stop hearing about events |
| Outbox publishing | Publishes the transactional outbox | Outbox rows accumulate; the unpublished-outbox metric rises; receivers hear nothing |

In wave 0a the workers additionally run as Compose services (13 services in total: 5 core services plus 8 workers), with `restart: unless-stopped` and direct database connections. The install script fails loudly, naming any worker not running, rather than starting silently with a missing worker.

Retention is CLI-only (`--once`, with an explicit `--apply` required before anything is deleted or anonymized; the default is a dry-run evaluation). The default is dry-run for a reason: a retention run deletes or anonymizes data, and the system refuses to do that on a default invocation. An operator must type the destructive flag deliberately.

## Observability — five series, and what they cannot tell you

Exactly five metric series are exposed: HTTP request totals, HTTP error totals, HTTP request duration, unpublished outbox count, and uptime seconds. No other series exist.

What those five can tell an operator: whether the system is up (uptime), whether it is erroring (error totals), whether it is slow (request duration), whether the outbox is draining (unpublished count), and how much traffic it is handling (request totals). What they cannot tell an operator: which tenant is slow, which endpoint is slow, which worker is stuck, how much money was collected today, or how many promises are due. Those are the questions an operator will ask first in a real incident, and the system cannot answer them from its metrics. That gap is stated here, not discovered in an incident.

Structured logs carry a trace identifier on every line; an incoming W3C `traceparent` header is accepted when it matches, otherwise an identifier is generated, and the effective identifier returns as a response header. Health and readiness are distinct: health is a static OK with no database touch; readiness runs a database query and checks the migration head, returning 503 with a checks object otherwise. A load balancer that checks health learns the process is alive; one that checks readiness learns the system can actually serve.

## Backup and restore — the drill, and its limits

The backup role is select-only: it bypasses row-level security for reads, which is exactly what a backup needs, and can write nothing.

The wave 0a restore drill, in full: a demo database of 168 tables and 10,000 invoices was backed up logically (1.8 seconds) and restored into a fresh database (71.4 seconds), for **about 73 seconds total against a under-15-minute target, with row counts matching after restore**. The drill passed with wide margin.

What the drill does not prove:

- That a backup taken under production load restores as fast (the drill ran on a quiet demo database).
- That an encrypted backup restores at all (backup encryption is not shipped; the mechanism is an owner decision).
- That a backup exists when needed (no backup schedule ships; RPO is unbounded by default — the system does not take backups on its own).
- That a restore preserves the audit chain's verifiability after a partial restore (the drill restored the whole database and compared row counts; it did not test a partial restore).

## Deployment — what exists, and what does not

**What exists:** a trial deployment on the owner's PC, loopback only, serving the demo tenant with fictional data. A Compose file that runs the API, the database, PgBouncer, and the 8 workers (13 services), with the web service excluded — the web service's boot in Compose is unproven, and the trial serves the web from a host process instead. Deployment manifests are written and reviewed.

**What does not exist:** a production deployment; live CI (the manifests are not proven in a live pipeline); production SLSA (none claimed); a status page; incident communications; per-tenant SLA measurement or reporting; product analytics; customer-success signals; load or soak testing on production-like infrastructure. The load figures in the evidence document were measured on a 2-vCPU demo VM and are not production capacity.

## What is deliberately not present

- **Frontend monitoring inventory.** No inventory of what the frontend monitors exists in code.
- **Application-log retention policy.** No retention policy for application logs is defined in code.
- **Operator alerting.** No alerting configuration exists. An operator is not paged when the outbox stops draining, when the error rate rises, or when a worker stops. The incident-response plan is drafted and undrilled, and there is no simulated exercise behind it.
- **Live CI, production SLSA, external penetration test, live KMS, live object-lock storage, hosted AI/OCR.** Each is either not run or not claimed; the security document lists them with their gates.
- **A `/verify-email` frontend page.** No new UX was built in 5.0.1, so verification completes through the API path only.
- **Per-tenant usage metering as a product surface.** Metering exists as a worker that records usage; there is no customer-facing usage page, and pricing without a metering surface is a commercial gap for the live money loop (see the roadmap).

Each absence is stated here so a reader does not assume it. The system's operations posture is: correct money handling and tenant isolation first, with the operational surface (alerting, status, SLA, analytics) deferred to Wave 1 — after the pilot proves the product, not before.

## Incident walkthrough — the outbox stops draining

A realistic incident, walked through with only the observability the system actually has:

- **Symptom an operator can see:** the unpublished-outbox metric rises, and stays up. External receivers stop hearing about events. Nothing else changes — HTTP error totals stay flat, request duration stays flat, uptime keeps counting.
- **What the operator cannot see:** which worker is stuck, which tenant's events are waiting, or how many events are affected per tenant — there is no per-tenant metric and no per-worker metric. The operator knows the outbox is not draining; they do not know why from the metrics alone.
- **What the operator does:** check whether the outbox publisher process is running (the install script fails loudly naming a missing worker, but a worker that dies *after* a successful start is not re-announced by the installer); check the structured logs for the publisher's trace identifiers; check the database for the oldest unpublished row and its age.
- **What does not happen automatically:** no alert fires (no alerting configuration exists), no page goes out, no failover starts. The incident is found by an operator watching the metric, not by the system raising its hand. That is the honest state of operations today, and it is why the incident-response plan being "drafted, undrilled" matters: the first real incident will be the first drill.

## Trial deployment topology — what runs where

The trial runs on the owner's PC, loopback only. The Compose topology is 13 services: 5 core services plus the 8 workers. The API sits behind PgBouncer; the workers connect directly to the database. The web service is excluded from Compose — its boot inside Compose is unproven, and the trial serves the web from a host process on the PC instead. There is no load balancer, no replica, no separate read database, and no staging environment: the trial is one machine, one database, one copy of each service. That topology is sufficient for a trial with fictional data and insufficient for production — which is why no production claim is made anywhere in this repository.
